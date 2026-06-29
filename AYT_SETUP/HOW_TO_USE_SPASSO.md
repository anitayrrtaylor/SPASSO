# How to Use SPASSO — Your Complete Guide

*Written for this Mac, where SPASSO is installed at `~/Desktop/SPASSO`. Last updated 2026-06-20.*

---

## 1. What SPASSO does

SPASSO downloads daily **satellite ocean data** (sea-surface height, currents, temperature,
salinity, chlorophyll) from **Copernicus Marine**, maps it over a region you choose, computes
oceanographic **diagnostics** (fronts, eddies, kinetic energy, Lagrangian transport), and bundles
everything into a **PDF bulletin** — optionally emailed to you. It's used to plan and interpret
research-cruise sampling.

You control *everything* through one text file: the **config file** for a "cruise".

---

## 2. Where everything lives

```
~/Desktop/SPASSO/
├── src/                         ← the program code (you run it from here)
│   └── Spasso.py                ← the main script you launch
├── Cruises/
│   └── WMedSeaExample/          ← one folder per "cruise" (a region + settings)
│       ├── config_WMedSeaExample.ini   ←★ THE FILE YOU EDIT (the control center)
│       ├── Figures/             ← output maps (.png)
│       ├── Bulletin/            ← output PDF + .tex bulletin
│       ├── Processed/           ← processed data files
│       ├── Logs/                ← run logs (check here if something fails)
│       └── Wrk/                 ← scratch/working files
├── Data/                        ← downloaded satellite data (auto-filled)
│   └── BATHY/                   ← bathymetry (seafloor depth) NetCDF
└── HOW_TO_USE_SPASSO.md         ← this guide
```

**A "cruise" = a folder in `Cruises/` + its `config_<Name>.ini` file.** You can have many.

---

## 3. The only command you need to run it

Open the **Terminal** app and paste this (it activates the right Python and runs the example):

```bash
cd ~/Desktop/SPASSO/src
PYTHONUTF8=1 /usr/local/Caskroom/miniforge/base/envs/test-env/bin/python Spasso.py WMedSeaExample
```

- Replace `WMedSeaExample` with your cruise's name to run a different one.
- `PYTHONUTF8=1` is required here because your Copernicus password contains special (Icelandic)
  characters — it makes Python read the config correctly.
- A full run takes a few minutes (it downloads data and computes diagnostics). When it finishes you'll
  see *"Successfully ended program… Thanks for using SPASSO."*

**Tip — make it a one-word command.** Run this once to create a shortcut called `spasso`:

```bash
echo 'spasso() { cd ~/Desktop/SPASSO/src && PYTHONUTF8=1 /usr/local/Caskroom/miniforge/base/envs/test-env/bin/python Spasso.py "$1"; }' >> ~/.zshrc
source ~/.zshrc
```
Then you can just type: `spasso WMedSeaExample`

---

## 4. The config file — your control center

Open it in any text editor (TextEdit is fine):

```bash
open -e ~/Desktop/SPASSO/Cruises/WMedSeaExample/config_WMedSeaExample.ini
```

The file is divided into `[sections]`. Each line is `setting = value`. Below are the edits you'll
actually make. **After editing, save the file and re-run the command in Section 3.**

> ⚠️ Keep the `=` alignment loose — spaces around `=` are fine. Don't add quotes around values.
> Lines starting with `#` are comments/examples (ignored).

---

### 4a. ✍️ Change the AUTHOR (on the PDF bulletin)

Find the `[bulletin]` section:

```ini
[bulletin]
authors             = John Doe
acknow              =
```

- Change `John Doe` to your name. Example: `authors = Anita Taylor`
- **Multiple authors?** Separate with commas: `authors = Anita Taylor,Louise Rousselet`
- `acknow` = an **optional acknowledgements section**. ⚠️ This is **not** the text itself — it's the
  **filename** of a text file (kept in this cruise's folder) whose contents get printed. To use it:
  create e.g. `acknowledgments.txt` in `Cruises/<Name>/`, type your text inside, and set
  `acknow = acknowledgments.txt`. **Leave `acknow` blank to skip it** (otherwise, if it names a file
  that doesn't exist, the bulletin step fails with `FileNotFoundError`).

> Note: the *"Contact: None"* you may see in the terminal is a different thing — that's the **email
> sender** (Section 4g), not the author.

---

### 4b. 🌍 Change the REGION (map area)

The region is set in **three** places — keep them consistent. Use decimal degrees
(negative = West / South).

```ini
[Eulerian]
loni                = -2, 10        ← longitude min, max
lati                = 40, 44.5      ← latitude min, max

[Lagrangian]
loni                = -2, 10
lati                = 40, 44.5

[cruise_param]
Lon                 = -2,10
Lat                 = 40, 44.5
```

Example — to map the Bay of Biscay you'd set roughly `loni = -10, 0` and `lati = 43, 48` in all three.

---

### 4c. 🛰️ Choose which DATA PRODUCTS to download

```ini
[products]
products           = PHY
# products         = PHY,SST_L4,SSS_L4,CHL_L4
```

Add the ones you want, comma-separated. Available product keys (defined further down in the file):

| Key | What it is |
|---|---|
| `PHY` | Sea-surface height + geostrophic currents (global) |
| `PHYEURO` | Same, higher-resolution European seas |
| `PHY_WIND` | Wind / wind-stress curl |
| `SST_L4` | Sea-surface temperature |
| `SSS_L4` | Sea-surface salinity |
| `CHL_L3` / `CHL_L4` | Chlorophyll-a (L3 = with gaps, L4 = gap-filled) |
| `MEDSEA_WAVF` | Mediterranean wave forecast |

Example: `products = PHY,SST_L4,CHL_L4`

> The same keys are also used in the `[Eulerian]`/`[Lagrangian]` `products =` lines to tell each
> diagnostic which field to use.

---

### 4d. 📅 Get TODAY's data vs. HISTORICAL dates (NRT vs DT)

```ini
[cruises]
cruise              = WMedSeaExample
mode                = NRT       ← NRT = near-real-time (most recent data, default)
refdate             =
dtmode              =
outmode             =
```

- **`mode = NRT`** → automatically uses the latest available data. Leave the other fields blank.
- **`mode = DT`** → you pick the dates. Fill in:
  ```ini
  mode      = DT
  refdate   = 2024-05-01,2024-05-15   ← dates in YYYY-MM-DD
  dtmode    = range                   ← "range" = every day between the two dates
                                        "ind"   = only the exact dates listed
  outmode   = daily                   ← "daily" = one figure per day
                                        "clim"  = an average over the period
  ```

---

### 4e. 📊 Choose which DIAGNOSTICS to compute

**Eulerian** (snapshot maps):
```ini
[Eulerian]
diag               = KE        ← KE = kinetic energy
# diag             = KE,OW     ← OW = Okubo–Weiss (eddy/front detector)
```

**Lagrangian** (particle-tracking transport):
```ini
[Lagrangian]
diag               = FTLE,TIMEFROMBATHY
```
Options you can list (comma-separated):

| Diagnostic | Meaning | Needs |
|---|---|---|
| `FTLE` | Finite-Time Lyapunov Exponents — fronts/filaments | velocity (PHY) |
| `LLADV` | Longitude/latitude advection | velocity |
| `OWTRAJ` | Okubo–Weiss along trajectories | velocity |
| `TIMEFROMBATHY` | Time since water last crossed a depth contour | **bathymetry file** (Section 5) |
| `SSTADV` | Advected sea-surface temperature | velocity + SST |

`numdays = 15` (further down) = how many days the particles are tracked. `mode = backward` tracks
where water came from; `forward` tracks where it's going.

---

### 4f. 🎨 Adjust the COLOR SCALES on maps

In `[plot_param]`, every product/diagnostic has `min`, `max`, `unit`. If a map looks washed-out or
saturated, tweak these. Examples:

```ini
kemin             = 0
kemax             = 1700          ← kinetic-energy color-bar top
sst_l4min         = 12
sst_l4max         = 17            ← temperature range in °C
ftlemax           = 0.4           ← FTLE color-bar top
```

---

### 4g. 📧 Email the bulletin automatically (optional)

```ini
[email]
sender_mail         = None        ← set to your "from" address, e.g. you@gmail.com
receiver_mail       =             ← who receives it (comma-separated for several)
smtp_server         =             ← your mail provider's SMTP server
port                =             ← e.g. 465 or 587
login               =             ← SMTP username
password            =             ← SMTP password / app-password
attach              = pdf,tex,tar ← what to attach
```
Leave `sender_mail = None` to **disable** emailing (it just saves the PDF locally). Gmail requires an
"app password", not your normal password.

---

### 4h. 🚩 Overlay stations, ship track, or a glider (optional)

```ini
[plot_options]
options             =             ← list any of: stations,waypoints,glider
outopt              =             ← "kml" to also export Google-Earth files

[stations]                        ← sampling-station dots
coordlon            = 5.23,5.36,5.61
coordlat            = 43.18,42.82,42.53
name                = 1,2,3

[waypoints]                       ← ship-track line
waylon              = 5.23,5.36,5.61
waylat              = 43.18,42.82,42.53
```
To show them, e.g.: `options = stations,waypoints`.

---

## 5. Bathymetry (seafloor depth) data

Already installed: `~/Desktop/SPASSO/Data/BATHY/ETOPO_2022_v1_30s_N90W180_bed.nc` (1.6 GB, global, NOAA
ETOPO 2022). It's **only needed for the `TIMEFROMBATHY` diagnostic** (Section 4e). The config line:

```ini
[Lagrangian]
bathyfile          = ETOPO_2022_v1_30s_N90W180_bed.nc
bathylvl           = -700      ← the depth contour to track, in metres (negative = below sea level)
```

If you ever need to re-download it:
```bash
curl -L -o ~/Desktop/SPASSO/Data/BATHY/ETOPO_2022_v1_30s_N90W180_bed.nc \
  "https://www.ngdc.noaa.gov/thredds/fileServer/global/ETOPO2022/30s/30s_bed_elev_netcdf/ETOPO_2022_v1_30s_N90W180_bed.nc"
```

---

## 6. Where the data comes from (Copernicus account)

All satellite data is free from **Copernicus Marine**. Your login is already saved in the config:

```ini
[userpwd]
userCMEMS          = ataylor12          ← your USERNAME (not your email)
pwdCMEMS           = (your password)
```

If you ever change your Copernicus password, update `pwdCMEMS` here. To check the login works:
```bash
/usr/local/Caskroom/miniforge/base/envs/test-env/bin/copernicusmarine login --username ataylor12
```

---

## 7. Create a brand-new cruise (your own region)

```bash
# 1. Make a new cruise folder (no spaces in the name is easiest)
mkdir ~/Desktop/SPASSO/Cruises/MyCruise

# 2. Copy the example config as a starting point
cp ~/Desktop/SPASSO/Cruises/WMedSeaExample/config_WMedSeaExample.ini \
   ~/Desktop/SPASSO/Cruises/MyCruise/config_MyCruise.ini

# 3. Edit it (region, products, author, etc. — Section 4)
open -e ~/Desktop/SPASSO/Cruises/MyCruise/config_MyCruise.ini

# 4. Run it
cd ~/Desktop/SPASSO/src
PYTHONUTF8=1 /usr/local/Caskroom/miniforge/base/envs/test-env/bin/python Spasso.py MyCruise
```

> The config filename **must** be `config_<FolderName>.ini` and match the name you pass on the command line.

---

## 8. Where your results appear

After a successful run, look in your cruise folder:

| Folder | Contents |
|---|---|
| `Cruises/<Name>/Figures/` | The map images (`.png`) |
| `Cruises/<Name>/Bulletin/` | The **PDF bulletin** (and `.tex` source) |
| `Cruises/<Name>/Logs/` | The run log — open this first if something went wrong |

Open the latest bulletin quickly:
```bash
open ~/Desktop/SPASSO/Cruises/WMedSeaExample/Bulletin/*.pdf
```

---

## 9. Troubleshooting

| Symptom | Fix |
|---|---|
| `Invalid credentials` | Username is `ataylor12` (not the email). Re-check `pwdCMEMS`. Reset password at marine.copernicus.eu if needed. |
| `interp2d has been removed` / SciPy error | The environment must keep **SciPy 1.13.1**. Run: `mamba install -n test-env scipy=1.13.1` |
| Password/accents read wrong | Always run with `PYTHONUTF8=1` (it's in the command in Section 3). |
| `TIMEFROMBATHY` fails | The bathymetry file (Section 5) must exist in `Data/BATHY/`. |
| `FileNotFoundError: …/<name>` at the **bulletin** step | Your `[bulletin] acknow` names a file that doesn't exist. Either create that text file in the cruise folder, or blank out `acknow`. |
| Run does nothing / exits after the package list | The cruise's config file must be named **exactly** `config_<FolderName>.ini` and its `[cruises] cruise =` line must match. |
| `KeyError: 'longitude'` during diagnostics (esp. on a re-run after a failed run) | Stale files left in `Wrk/` by a previous **failed** run. SPASSO only auto-cleans `Wrk/` after a *successful* run. Fix: empty the work folder, then re-run (see below). |

**If a run fails partway, clear the work folder before retrying** (replace the cruise name):
```bash
rm -f ~/Desktop/SPASSO/Cruises/KeflavikExample/Wrk/*.nc \
      ~/Desktop/SPASSO/Cruises/KeflavikExample/Wrk/*.png \
      ~/Desktop/SPASSO/Cruises/KeflavikExample/Wrk/*.tex
```
| A map is all one color | Adjust that product's `min`/`max` in `[plot_param]` (Section 4f). |
| Nothing downloads / network error | Check internet; check the `Logs/` file for the exact failing product. |
| General failure | Open the newest file in `Cruises/<Name>/Logs/` — it shows exactly which step failed. |

---

## 10. Quick cheat-sheet

```bash
# Run the example
cd ~/Desktop/SPASSO/src
PYTHONUTF8=1 /usr/local/Caskroom/miniforge/base/envs/test-env/bin/python Spasso.py WMedSeaExample

# Edit settings
open -e ~/Desktop/SPASSO/Cruises/WMedSeaExample/config_WMedSeaExample.ini

# View the result
open ~/Desktop/SPASSO/Cruises/WMedSeaExample/Bulletin/*.pdf
```

| I want to… | Edit this in the config |
|---|---|
| Change the author | `[bulletin] authors` |
| Change the map region | `[Eulerian]`/`[Lagrangian]` `loni`,`lati` + `[cruise_param]` `Lon`,`Lat` |
| Pick data types | `[products] products` |
| Use past dates | `[cruises] mode = DT` + `refdate`,`dtmode`,`outmode` |
| Pick diagnostics | `[Eulerian] diag`, `[Lagrangian] diag` |
| Fix map colors | `[plot_param]` `…min`/`…max` |
| Email results | `[email]` section |
| Add stations/track | `[plot_options]` + `[stations]`/`[waypoints]` |
| Update your password | `[userpwd] pwdCMEMS` |
```
