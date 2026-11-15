# SPHEREview

**Blink the infrared sky with NASA SPHEREx. Type a star's name and watch the sky move.**

SPHEREview is a web-tool prototype for the NASA Space Apps Challenge 2026 challenge
**"Planet X and SPHEREx"**. Enter coordinates or an object name. SPHEREview plays the SPHEREx images
of that patch of sky like a little movie, so users can inspect stars, brown dwarfs, asteroids and
variable stars that move or change brightness. Because SPHEREx sees the sky in 102 infrared colours,
the tool also turns any point into a spectrum on the fly.

The application runs locally from this repository. A public deployment URL has not been added yet.
SPHEREview is an exploration and visualization tool; it does not automatically scan the whole sky
or confirm new planet discoveries.

![Barnard's Star blinking in SPHEREx band D6, with Gaia DR3 positions moved to each frame's date](docs/screenshots/01_barnard_D6_gaia.jpg)

## New: three workspaces, discovery reports and guides

After choosing a target, the top bar switches between three workspaces:

- **Image subtraction** (`/api/subtraction`). Downloads at least 10 (10/12/16/24) individual SPHEREx
  Level 2 exposures of the field from NASA/IPAC IRSA. Exposures are chosen so that as many as possible
  form *wavelength-matched pairs from different dates* (SPHEREx's linear variable filters make every
  exposure see a slightly different wavelength). Any two can be subtracted (B − k·A, or B minus a
  median template of the other matched exposures); constant stars cancel, and changes glow orange
  (brighter/new) or blue (fainter/gone), with pulsing rings and arrows for moved sources.
  Significance uses the SPHEREx VARIANCE layer, re-calibrated on the image.
- **Infrared bands** (`/api/bands`). All 102 SPHEREx channels of the field from the DR1 spectral cube.
  Presets (detectors, short/mid/long, water ice, methane/PAH, CO₂, CO, cold-object window) or
  click/shift-click channels; inverse-variance stacking, RGB composites or single channels; an S/N meter
  compares the measured gain with √N; click a pixel for its 102-point spectrum.
- **Blink player** — the original viewer.

Other additions:

- **Discovery reports** (`/api/reports`). “Report discovery” (or ✚ Report on a detected change or a star's
  pop-up) saves a *New Candidate Object* with coordinates, compared exposures, a snapshot and automatic
  Gaia DR3 / NASA JPL small-body checks. Reports appear on the shared **Community** board, where others can
  confirm them; only the author's browser holds the token needed to delete one. Stored in
  `backend/data/community_reports.json`.
- **Solar System presets** (`/api/solar-system`). Planets, dwarf planets and asteroids: NASA JPL Horizons
  gives the body's position *as seen from the SPHEREx spacecraft*, and IRSA SIA finds the exposures that
  actually contained it; the field is centred on that visit.
- **Exposure index fallback.** IRSA SIA v2 is used first (seconds), IRSA TAP second.
- **Quick tour** on first start (six illustrated pages with call-outs) and full-screen **User guide** and
  **Science & methods** pages in the ··· menu. The tour illustrations are real SPHEREx data rendered by
  `scripts/make_tour_images.py`.
- The whole interface is in English.

## What it does

- **Blink any patch of sky.** Enter RA/Dec (decimal or sexagesimal) or a name, and choose a 3′–15′ field.
  Frames stream in as they are built, so the first band appears before the rest has loaded.
- **Use the 102 bands.**
  - Per-detector images (D1–D6, 0.75–5.0 µm).
  - An all-band signal-to-noise image.
  - An RGB false-colour composite in which cold, red objects stand out.
- **Add or subtract epochs.**
  - *Difference* mode subtracts the per-pixel median of all epochs, so moving sources show up as orange/blue pairs.
  - *Sum* adds every epoch into one deeper image.
- **Overlay other telescopes.**
  - **Gaia DR3** (ESA, via MAST) and **CatWISE2020** (NASA WISE/NEOWISE) sources are moved with their proper motions to each frame's date, so the markers travel with the stars.
  - Fast movers are listed for the field. Named targets get a reticle at their predicted position.
- **Spectra on the fly.** Click a source to get its 0.75–5 µm spectrum.
  - Each SPHEREx exposure sees a source at a different wavelength, so aperture photometry across exposures gives the spectrum.
  - Molecular bands (H₂O, CH₄, CO₂, CO, PAH) are marked.
- **Light curves.** Brightness per epoch and detector for any source, computed in the browser.
- **Measured position changes.** Select a source and open *Position* to plot its image centroid offsets by date; detector measurements are combined by their epoch. Pixel sampling, blending and image quality limit the astrometric precision.
- **Metadata.** Each frame shows its date, wavelength range, number of images, data release and contamination flag counts. Flagged spectral points are drawn hollow.
- **Shareable and embeddable.** Every setting lives in the URL. `?embed=1` shows only the player. A button exports an animated GIF.
- **Sky context.** An Aladin Lite panel shows the same field in 2MASS, AllWISE, the SPHEREx all-sky cube, SDSS and DSS2.
- **Configured for new releases.** Add the archive collection and storage prefix in [`config/releases.yaml`](config/releases.yaml); image coverage is queried live from IRSA.

## The challenge targets

| Target | What you see in SPHEREview |
|---|---|
| **Barnard's Star** | Moves ~10.4″/yr: 1.9 px north over three epochs (Aug 2025 – Aug 2026). The Gaia DR3 ring follows it, and Difference mode shows a strong dipole. |
| **Epsilon Indi B** | The T-dwarf pair is bright at 4.4–5 µm. Four epochs, with the Gaia ring (G = 18) and the predicted-position reticle on top of it. |
| **WISE 0855−0714** | The coldest known brown dwarf: invisible to Gaia, faint but visible in D6. CatWISE lists it moving 8.0″/yr with W1−W2 = 3.8. Its spectrum shows the 3.3 µm methane trough. Note: the challenge's coordinates date from ~2010, and in 2025–26 the object is ~2′ west of them. SPHEREview's "Fast movers" list and the CatWISE overlay find it anyway. |
| **RW Aurigae** | Light curve: the 3.8–5 µm flux drops from ~2.7 Jy to ~1.3 Jy across three epochs. |
| **R Draconis** | Repeated SPHEREx deep-field observations provide a time series for examining this Mira variable's changing brightness. |

<p>
<img src="docs/screenshots/03_wise0855_catwise.jpg" width="49%" alt="WISE 0855 with CatWISE overlay">
<img src="docs/screenshots/05_r_dra_light_curve.jpg" width="49%" alt="R Draconis light curve">
</p>

## How it works

```
Browser (HTML/JS, no build step)                     FastAPI backend (Python 3.12)
  search · blink player · overlays · plots  ◀─NDJSON─  /api/frames   /api/spectrum
  stretch + colour maps computed in JS                 /api/resolve  /api/gaia  /api/catwise
                                                         │
          IRSA TAP (which images cover this point?) ◀────┤
          AWS S3 nasa-irsa-spherex (HTTP Range reads) ◀──┤  ~0.5 MB per image instead of 63 MB
          MAST Mast.Catalogs.GaiaDR3.Cone  ◀─────────────┤
          IRSA Gator (CatWISE2020), CDS Sesame ◀─────────┘  everything cached on disk
```

1. **Find images.** The backend runs one IRSA TAP query that joins `spherex.plane`, `spherex.artifact`
   and `spherex.observation`. It returns every SPHEREx DR1 + QR3 Level 2 image covering the position.
   Images are grouped into epochs by time gaps. Deep fields near the ecliptic poles, observed almost
   continuously, are cut into ~30-day bins.
2. **Read only the pixels needed.** A small FITS reader issues HTTP Range requests against the public
   S3 copy. It reads the header, the needed rows of the `IMAGE` HDU, and the needed rows of the
   RICE-compressed `FLAGS` HDU, decoding one tile per row. That takes 0.8 s and ~0.5 MB per image.
   Reading through astropy/fsspec took ~10 s per image, and the IRSA cutout service returns ~2 MB
   per image.
3. **Coadd.** For each epoch and detector, up to four exposures are background-subtracted and
   dead pixels are masked. Saturated cores are kept visible. The exposures are reprojected onto a
   common north-up grid at the native 6.15″ pixel scale and median-combined.
4. **Stream.** Frames are sent as small float32 arrays (~12 kB each) in newline-delimited JSON as soon
   as they are ready. The browser does the stretch and colour mapping, so the sliders respond instantly.
5. **Catalogues.** Gaia DR3 and CatWISE positions are propagated from their reference epochs
   (J2016.0 and MJD 57170) to each frame's date. Names are resolved with CDS Sesame and moved to 2026.0.
6. **Spectra.** For a clicked source, the backend does aperture photometry on each exposure (photutils,
   r = 2 px with a local background annulus). Each measurement is tagged with the wavelength the detector
   saw at that pixel, from the spectral WCS (`WCS-WAVE`). That table is cached once per detector.

The decisions behind each step, with measurements, are in [docs/spike_report.md](docs/spike_report.md).
Literature and method review, including the A-JINX Chrono & Spatial project, is in [docs/literature_review_tr.md](docs/literature_review_tr.md). The Turkish control/audit report for the AI panel and interface is in [docs/kontrol_raporu_tr.md](docs/kontrol_raporu_tr.md). In Difference mode the backend also reports ≥5σ changes from wavelength-matched epochs, overlays them as magenta squares, and checks candidates against Gaia DR3 and JPL SB Identification. Repeat detections are associated across epochs to build motion tracks; a second pass groups brightening/fading pairs at the same position into variability tracks. The AI-assisted panel is a transparent evidence model (v2), not a trained classifier: it combines epoch coverage, linear-motion consistency, detection significance, flux-sign consistency, Gaia/JPL cross-matches and image-artifact context (field edge, opposite-polarity dipole, crowding). Every track exposes its signed contributions, a confidence tier and a recommended next step; known catalogue matches and likely artifacts are deprioritised and can be filtered separately. Unconfirmed tracks with fewer than three epochs are limited to low confidence because two points cannot test linearity. Candidates are leads for review, not confirmed discoveries; the JPL check uses an Earth-geocentre approximation because SPHEREx state vectors are not included here.

Candidate measurements and the viewer's processed image cutouts are automatically kept in the browser's local archive (last 25 searches) and can be reopened or imported/exported as JSON/CSV. Position and brightness plots offer raw, constant, linear and quadratic models; “best fit” selects the lowest BIC for the available measurements and reports R²/RMSE. Additional filters include minimum score, epochs, significance and maximum fit residual. The original full FITS files remain in the NASA archive.

## Run it

Requires Python 3.12 and an internet connection for the first package install and NASA/catalog data. No API keys are needed.

On Windows, double-click [`BASLAT.bat`](BASLAT.bat) (or [`RUN_SPHEREVIEW.bat`](RUN_SPHEREVIEW.bat)); on macOS/Linux, run `sh BASLAT.sh`. The launcher creates a local `.venv`, installs the pinned packages once, selects a free port from 8000–8020 and opens the browser. Do not open `frontend/index.html` directly: the page needs the local `/api` backend.

For a portable source package, run `python scripts/package_portable.py`. This creates `dist/SPHEREview-portable.zip`, excluding Python environments, downloaded caches and local FITS data. See [`README_RUN_TR.md`](README_RUN_TR.md) for setup on another computer.

The launcher opens the browser; if it does not, open the local URL printed in its terminal. Try `Barnard's star`, `WISE 0855` or `R Dra`.

- **Warm the cache before a demo:** `.venv\Scripts\python scripts\warm_cache.py` pre-loads the five challenge targets.
- **Tests:** `.venv\Scripts\python -m pytest`. 49 tests, including the Range reader against a synthetic SPHEREx-like file and the evidence model's motion/artifact/variability branches.
- **Docker / Hugging Face Spaces:** see [docs/deploy.md](docs/deploy.md).
- **First loads:** the first load of a new field downloads ~0.5 MB per image. Hosting the backend
  near AWS us-east-1 makes this much faster than a home connection. Browsers only receive the small frames.

## API

| Endpoint | Returns |
|---|---|
| `GET /api/resolve?q=` | Field centre for coordinates or a name, including proper motion |
| `GET /api/frames?ra=&dec=&fov=&det=all\|1,2,…&mode=epoch\|exposure` | NDJSON: `meta`, then frames, then `done` |
| `GET /api/spectrum?ra=&dec=&field_ra=&field_dec=&mode=quick\|full[&epoch=&pmra=&pmdec=&ref_mjd=]` | NDJSON spectral points (mJy, µm, flags) |
| `GET /api/gaia?ra=&dec=&fov=` | Gaia DR3 sources (J2016.0) |
| `GET /api/catwise?ra=&dec=&fov=` | CatWISE2020 sources (MJD 57170) |
| `GET /api/observations?ra=&dec=` | Every exposure covering a point, with epoch labels |
| `POST /api/export/gif` | Animated GIF from frames the browser rendered |

## Project layout

```
backend/   app.py · spherex/ (tap, fits_range, cutouts, wave, coadd, epochs, frames, photometry, spectrum)
           catalogs/ (gaia, catwise, resolve)
frontend/  index.html · app.js · render.js · sky.js · overlay.js · analysis.js · context.js · styles.css
config/    releases.yaml (data releases, viewer settings) · targets.yaml (challenge targets)
scripts/   warm_cache.py      tests/   pytest suite      spikes/   phase-1 data experiments
docs/      spike_report.md · deploy.md · submission.md · demo.md · screenshots/
```

## Limitations and next steps

- **Motion resolution.** Pixels are 6.15″ and the current baseline is ~1–1.4 years, so only motions
  above ~1″/yr show clearly by eye. Difference mode and the catalogue overlays help with the rest.
  The baseline grows with every new release.
- **Blends.** Sources closer than ~12″ blend in the photometry. For example, WISE 0855 currently overlaps
  a background star at short wavelengths.
- **Current limitations:**
  - a per-pixel flag overlay (flags are summarised per frame and on spectral points)
  - solar-system objects by name (JPL Horizons)
  - a public deployment close to the data; the app is currently run locally

## Use of Super Intelligence (SI)

This project was built with **Claude (Anthropic)**, an SI coding assistant, working under the team's
direction. This was a 48-hour hackathon with SI-assisted engineering. The roadmap and architecture were drafted
with SI from the challenge statement's objectives and considerations. Most of the source code in
`backend/`, `frontend/`, `scripts/`, `spikes/` and `tests/`, and the documentation in `docs/` and this
README, were generated with SI assistance. The team chose the challenge and steered the work. The
results were tested with the included unit tests and checked against the real archive data.
Screenshots in `docs/screenshots/` are captures of the running app showing real SPHEREx data; no
images were generated. See [SI_DISCLOSURE.md](SI_DISCLOSURE.md).

## Data and acknowledgements

- **SPHEREx** DR1 and QR3 spectral images from the NASA/IPAC Infrared Science Archive (IRSA).
  DOIs: 10.26131/IRSA678 (DR1), 10.26131/IRSA662 (QR3), 10.26131/IRSA679 (DR1 all-sky cube).
  *This publication makes use of data products from the Spectro-Photometer for the History of the
  Universe, Epoch of Reionization and Ices Explorer (SPHEREx), which is a joint project of the Jet
  Propulsion Laboratory and the California Institute of Technology, and is funded by the National
  Aeronautics and Space Administration.*
- **Gaia DR3** (ESA) via MAST, with the ESA Gaia Archive as fallback. *This work has made use of data
  from the European Space Agency (ESA) mission Gaia (https://www.cosmos.esa.int/gaia), processed by the
  Gaia Data Processing and Analysis Consortium (DPAC).*
- **CatWISE2020** (Marocco et al. 2021, ApJS 253, 8) via IRSA. It is based on WISE and NEOWISE, a joint project
  of UCLA and JPL/Caltech funded by NASA.
- **2MASS**, AllWISE, SDSS and DSS2 HiPS mirrored at IRSA. **SIMBAD/Sesame** and **Aladin Lite**
  (Boch & Fernique 2014) by CDS, Strasbourg.
- Built with astropy, astroquery, pyvo, reproject, photutils, numpy, FastAPI and Pillow.
- Inspired by [WiseView](http://byw.tools/wiseview) (Caselden et al., ascl:1806.004) and the Backyard Worlds: Planet 9 project.

Team name and member names should be entered on the NASA Space Apps project page.
