# Artemis Lunar Landing Site Navigator

**NASA Space Apps Challenge 2026**  
**Architecture:** Vanilla JavaScript ES Modules, Three.js WebGL, Astronomy Engine, HTML5 Canvas 2D.  
**Terrain Status:** Real LOLA DEM (NASA PGDA `LDEM_80S_80MPP_ADJ`, resampled to 320 m); synthetic procedural terrain is the automatic fallback if the files fail to load.

---

## 1. Overview

The **Artemis Lunar Landing Site Navigator** is an interactive 3D WebGL and scientific computing tool modeling the Sun-Earth-Moon orbital system and lunar south-polar terrain. It is designed to perform geometry-grounded evaluations of NASA’s nine candidate Artemis III landing regions (announced October 2024).

### Key Technical Capabilities:
- **Zero-Build Vanilla ES Modules**: Runs natively in any standard modern web browser using standard `<script type="importmap">` mapping to pinned ES modules on `cdn.jsdelivr.net/npm`.
- **Three.js Orrery & Polar Terrain**: Dual camera modes:
  1. Sun-Earth-Moon orbital orrery with realistic axial tilts, orbit paths, and sub-solar vector lines.
  2. South-polar stereographic terrain mesh (-84° to -90°S) with dynamic real-time solar shading.
- **Topocentric Ephemeris Engine**:
  * Sun & Earth directions in the Moon-fixed selenographic frame using `astronomy-engine`.
  * Topocentric surface parallax correction: $\vec{d}_{\oplus} = D_{\oplus}\hat{r}_{\oplus} - \vec{r}_{\text{obs}}$ (shifts apparent Earth elevation by up to $\sim 0.268^\circ$).
- **Topographic Horizon Ray-Marching**:
  * 720 azimuth bins (0.5° angular resolution) computed over terrain heightfield accounting for lunar spherical curvature.
  * Cached in-memory per region for zero per-frame overhead.
- **Extended-Disc Occlusion Quadrature**:
  * 64-strip midpoint consistent quadrature integrating Sun (0.53° dia) and Earth (2.0° dia) discs against terrain horizon masks.
  * Analytical circular segment formula for boundary verification.
- **Direct Line-of-Sight Comms Ray**: 3D vector directed from selected landing site toward Earth, dynamically color-coded by unblocked Earth fraction ($f_{\oplus} \ge 0.95$ Cyan, $f_{\oplus} \le 0.0$ Coral, linear interpolation for partial occultation).
- **Libration Elevation Chart**: Pure Canvas 2D time-series plot depicting Earth's $\sim 27.2$-day draconic oscillation and terrain occultation threshold across the selected time span.
- **Multi-Criteria Region Ranking**: Configurable weighted trade-study slider scoring solar illumination, Earth visibility, and terrain slope flatness.
- **Automated Verification Suite**: Unified test suite runnable in browser (`tests.html`) and headless in Node.js (`tests/run.mjs` / `npm test`).

---

## 2. Running Locally

The application is a pure static site utilizing native browser ES modules and pre-compiled static stylesheets.

### Option A: Static HTTP Server (Recommended)
```bash
npm run serve
```
Serves on `http://localhost:3000` using Node's built-in HTTP server (`serve.js`).

### Option B: Build Static Stylesheet (Tailwind CSS v3 CLI)
```bash
npm run build:css
```
Compiles and minifies `src/tailwind.input.css` into `src/tailwind.generated.css` using the pinned `tailwindcss@3.4.17` CLI. (Pre-compiled in the repository; no build step required to run the site).

### Option C: Automated Verification Test Suite
```bash
npm test
```
Runs the headless test runner (`tests/run.mjs`) verifying all 14 acceptance criteria against 2D brute-force numerical integration, CSS class completeness audit, and JSDOM panel visibility logic.

You can also view the test suite in a browser by navigating to `http://localhost:3000/tests.html`.

---

## 3. Topography Provenance & Real LOLA Ingestion

The app loads `public/data/lola_south_pole_320m.bin` + `.json` at start-up (paths in `config.js` → `dem`). It is NASA PGDA's `LDEM_80S_80MPP_ADJ` (https://pgda.gsfc.nasa.gov/products/90; cite Barker et al. 2023, PSJ 4, 183), block-averaged 4×4. If the files cannot be fetched or fail validation, the app runs on **synthetic placeholder terrain** and shows the banner *“Illustrative terrain – not real LOLA data”*.

To regenerate or change the resolution (needs `numpy tifffile imagecodecs`):
```bash
python3 scripts/convert_lola_dem.py path/to/LDEM_80S_80MPP_ADJ.tiff 4   # 4 -> 320 m; 2 -> 160 m (58 MB)
```
The script writes the Float32 raster (km above the 1737.4 km sphere), the metadata JSON with SHA-256, and `npm test` (Test 17) re-checks integrity and orientation. Set `dem.fallbackToSynthetic: false` to make a missing DEM a hard error.

---

## 4. Source Architecture

```
.
├── config.js               # Global configuration (radii, design tokens, pinned versions)
├── regions.js              # 9 candidate regions metadata (status: "pending")
├── index.html              # Entry HTML shell with pinned importmap and module loader
├── tests.html              # Browser verification test suite UI
├── serve.js                # Lightweight zero-dependency static server on port 3000
├── package.json            # Pinned dependencies (astronomy-engine, three, tailwindcss, jsdom) & scripts
├── tailwind.config.cjs     # Tailwind CSS v3 CLI configuration (preflight: false)
├── README.md               # Architecture documentation and running instructions
├── DATA_AND_METHODS.md     # Mathematical derivations & physical equations
├── NEEDS_HUMAN_VERIFICATION.md # Audit checklist of items pending human verification
├── CHANGELOG.md            # Version changelog with before/after audit numbers
├── tests/
│   ├── suite.js            # Unified test suite shared between Node and browser (14 tests)
│   ├── run.mjs             # Headless CLI runner (node tests/run.mjs)
│   ├── classAudit.js       # CSS class completeness audit regression test
│   └── panelVisibility.js  # JSDOM panel visibility & UI shell logic test
└── src/
    ├── app.js              # Application bootstrap & coordinator
    ├── ui.js               # UI components, HUD, dock controls, comparison table
    ├── scene.js            # Three.js 3D viewport, terrain mesh, orrery & comms ray
    ├── ephemeris.js        # Moon-fixed celestial vectors & topocentric parallax
    ├── horizon.js          # Ray marching & 64-strip midpoint disc fraction quadrature
    ├── terrain.js          # Polar stereographic coordinates & localized slope engine
    ├── demLoader.js        # DemProvider interface & synthetic polar terrain generator
    ├── comparisonEngine.js # Multi-region annual illumination & weighted ranking
    ├── earthElevationChart.js # HTML5 Canvas 2D libration time-series chart
    ├── timeStore.js        # Central simulation clock with delta-time integration
    ├── dataAndMethodsDoc.js# In-app viewer for DATA_AND_METHODS.md
    ├── tailwind.input.css  # Tailwind input stylesheet (@tailwind base; @tailwind utilities;)
    ├── tailwind.generated.css # Pre-compiled minified utility stylesheet
    └── index.css           # Standalone theme & component CSS stylesheet
```
