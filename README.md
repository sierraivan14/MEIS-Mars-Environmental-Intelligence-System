# MEIS — Mars Environmental Intelligence System (`ARES-PATH` / `HORIZON.exe`)

[![NASA Space Apps 2026](https://img.shields.io/badge/NASA_Space_Apps_2026-Galactic_Problem_Solver-0B3D91?style=for-the-badge&logo=nasa&logoColor=white)](https://www.spaceappschallenge.org/)
[![Live Demo](https://img.shields.io/badge/Live_Demo-Render-46E3B7?style=for-the-badge&logo=render&logoColor=0F172A)](https://meis.onrender.com/)
[![Python](https://img.shields.io/badge/Python-3.10%2B_(Stdlib)-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![SQLite](https://img.shields.io/badge/SQLite-marte__telemetria.db-003B57?style=for-the-badge&logo=sqlite&logoColor=white)](https://www.sqlite.org/)

> **Galactic Problem Solver — NASA Space Apps Challenge 2026 (UNAQ Querétaro, Mexico)**
>
> **Live Web Application:** **[https://meis.onrender.com/](https://meis.onrender.com/)**

MEIS (`ARES-PATH` / `HORIZON.exe`) is an interactive Martian tactical navigation and astronaut life-support telemetry platform. It combines real planetary imagery and environmental datasets with a graph-based **A\*** pathfinding engine and physiological **EVA / PLSS** (Extravehicular Activity / Portable Life Support System) resource modeling to plan safe surface traverses across Mars.

---

## 1. The Martian Challenge & Our Solution

### Problem
On Mars, surface astronauts and rovers operate without GPS, paved infrastructure, or breathable atmosphere. Traverses spanning regional basins and volcanic plateaus face severe hazards:
- Steep tectonic fractures, canyons, and shield volcanoes (*Valles Marineris*, *Noctis Labyrinthus*, *Erebus Montes*, *Olympus Mons*, *Hellas Planitia*).
- Loose regolith slopes and landslide corridors (*Lycus Sulci*, *Acheron Fossae*) that degrade traction and spike metabolic expenditure.
- Sudden local and regional dust storms that drop ambient temperature, reduce visibility, and force emergency detours.
- Strict oxygen and electrical power budgets on the spacesuit's PLSS backpack.

### Solution
**MEIS** acts as a mission-critical Martian tactical navigator:
1. **Visualizes Mars in 3D and 2D** using official planetary mosaic tiles (`EPSG:4326` / Mars Trek `EPSG::104905`) and zoom-progressive IAU nomenclature (`992` geological features).
2. **Computes hazard-aware A\* routes** around circular obstacle boundaries using 24-point tangent rings, avoiding strictly prohibited terrain while penalizing caution corridors.
3. **Models real-time EVA/PLSS vital consumption** (`O2` and battery `Wh_e`) from `100%` at safe zones, triggering blinking low-reserve evacuation (`<20%`) to the nearest certified refuge (`es_refugio = 1`), recharging to `100%`, and resuming the primary mission objective once safe.

---

## 2. Technical Architecture

```text
+-------------------------------------------------------------------+
|  Frontend (Archivo1.html)                                         |
|  - 3D Orbital Mars Globe (Three.js + Globe.gl, Viking + MOLA)     |
|  - 2D Tactical Map (Leaflet L.CRS.EPSG4326 + NASA Mars Trek WMTS) |
|  - Responsive EVA/PLSS HUD, Live Dust Storms & Reserve Rerouting  |
+---------------------------------+---------------------------------+
                                  | REST / JSON (HTTP)
                                  v
+-------------------------------------------------------------------+
|  Backend API Server (servidor_api.py)                             |
|  - Zero-dependency Python HTTPServer (port 8000)                  |
|  - GET /api/datos-mapa | POST /api/ruta-optima | POST /api/emerg..|
+---------------------------------+---------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|  Pathfinding & Physiology Engine (motor_astar.py)                 |
|  - A* Search on 24-Vertex Circular Tangent Rings (1.07 * R)       |
|  - Martian Spherical Scale Distance (R = 3389.5 km, 59.158 km/deg)|
|  - EVA/PLSS Metabolic, O2 (Weir) & Thermal/Power Estimators       |
+---------------------------------+---------------------------------+
                                  | sqlite3
                                  v
+-------------------------------------------------------------------+
|  Persistent Telemetry Database (marte_telemetria.db)              |
|  - ubicaciones_principales (6)   | zonas_obstaculos (7)           |
|  - perfiles_regionales_json (4)  | parametros_atmosfericos (1)    |
|  - nomenclatura_marte (992 IAU planetary features >= 30 km)       |
+-------------------------------------------------------------------+
```

### Core Subsystems

#### A. Tactical A\* Navigation Engine (`ARES-PATH` — `motor_astar.py`)
- **Tangent-Ring Visibility Graph:** Rather than rasterizing the entire planet into a memory-heavy fine grid, `construir_grafo_navegacion()` generates **24 perimeter waypoints** (`15°` intervals at radius `1.07 * R`) around each geological obstacle and active dust storm.
- **Collision & Rule Enforcement (`evaluar_interseccion_tramo`):** Samples `50` points along each candidate edge against obstacle radii (`1.05 * R`):
  - **`Esquivar` (Mandatory Avoidance):** *Valles Marineris*, *Noctis Labyrinthus*, *Erebus Montes*, plus *Olympus Mons* and *Hellas Planitia* (dynamically unblocked only when explicitly selected as mission origin or destination).
  - **`Precaucion` (Penalized Corridors):** *Lycus Sulci* and *Acheron Fossae* apply high regolith slip (`eta = 1.8`) and generate caution telemetry warnings.
- **Dynamic Emergency Rerouting (`desviar_a_refugio_mas_seguro`):** Under active dust storms or critical `<20%` suit reserves, A\* filters strictly for safe camp locations (`es_refugio = 1`: *Arcadia Planitia*, *Amazonis Planitia*, *Meridiani Planum*, *Utopia Planitia*), excluding non-rechargeable extreme sites (*Olympus Mons*, *Hellas Planitia*).
- **Planetary Distance Geometry (`calcular_distancia_marte_km`):** Evaluates non-wrapping geodesic segments via 10-step trapezoidal integration scaled to Mars's volumetric mean radius (`R = 3389.5 km`, `59.158 km/deg`).

#### B. SQLite Telemetry & Persistence (`marte_telemetria.db`)
All spatial sites, hazard zones, atmospheric constants, and nomenclature reside locally in SQLite for deterministic, microsecond reads and zero external database infrastructure:
- `ubicaciones_principales` (`6` rows): Safe camps & target destinations with central coordinates, diameter, temperature (`K`), pressure (`Pa`), elevation (`m`), slope (`deg`), and baseline risk.
- `zonas_obstaculos` (`7` rows): `5` mandatory avoidance hitboxes and `2` caution zones.
- `perfiles_regionales_json` (`4` rows): Regional risk and environmental limitation profiles.
- `parametros_atmosfericos` (`1` row): Martian ambient baseline (`T_nom = -73.15 °C`, `T_tormenta = -123.15 °C`, `P_nom = 840 Pa`, `g = 3.711 m/s²`).
- `nomenclatura_marte` (`992` rows): Curated IAU planetary nomenclature (`diameter >= 30 km`) sorted descending for zoom-level decluttering.

#### C. EVA / PLSS Life-Support Telemetry Model (`motor_astar.py`)
Calculates physiological and suit load per traverse segment (`calcular_rendimiento_y_costo_tramo`):
- **Effective Traverse Speed:** Baseline `v0 = 0.85 m/s`, scaled by regolith slip (`eta`), terrain slope (`max(0.45, 1 - 0.022 * |slope|)`), and storm penalty (`0.85x`), floored at `0.28 m/s`.
- **Metabolic Power (`M_dot_EVA`):** Adapted from **Pandolf (1977)** to Martian gravity (`gamma_g = 0.3783`, `f_g = 0.6892`) with pressurized suit stiffness factor **`k_suit = 1.5`** (Norcross, 2010) and low-pressure compensation for `m_b = 80 kg` + `m_s = 65 kg` (`145 kg` total), standing baseline `~192.4 W_th` and capped at `500 W_th`.
- **Oxygen Consumption:** Derived via **Weir's equation** (`0.2527 g/Wh_th`) plus `8.0 g/h` PLSS structural leakage against a **`950.0 g` (`665.5 L STP`)** primary O2 tank.
- **PLSS Electrical & Thermal Budget:** Combines `95 W_e` base avionics/LSS/LCVG, cooling pump & amine scrubbers, cold-environment heating (`0.65 * max(0, 240 - T_K)`), and dust load against a **`1100.0 Wh_e` (`1.10 kWh_e`)** battery.

#### D. Scientific & Geospatial Data Integration (NASA / ESA)
- **Topography & Surface Color Imagery:** Integrated with **NASA Mars Trek** WMTS layers (`Mars_Viking_MDIM21_ClrMosaic_global_232m` and **MGS MOLA** shaded relief `Mars_MGS_MOLA_ClrShade_merge_global_463m`) in native `EPSG:4326` (`EPSG::104905`).
- **Planetary & Environmental Datasets:** Regional parameters, slope hazard profiles, thermal inertia, and dust-storm dynamics informed by **MOLA** (Mars Orbiter Laser Altimeter), **HRSC** (ESA High Resolution Stereo Camera), **THEMIS** (Thermal Emission Imaging System), **Viking Lander 2**, **Zhurong**, **Opportunity**, and IAU `combinedNom` planetary nomenclature.

---

## 3. Tech Stack & Repository Structure

| Layer | Technology | Purpose |
| :--- | :--- | :--- |
| **Backend & Routing** | Python 3 Standard Library (`http.server`, `heapq`, `math`, `json`) | Zero-dependency REST API & A\* pathfinding engine |
| **Database** | SQLite 3 (`sqlite3`) | Local relational persistence for telemetry, hazards & 992 IAU features |
| **3D / 2D Visualization** | HTML5, CSS3, JavaScript (ES6+), Three.js, Globe.gl, Leaflet `1.9.4` | Responsive 3D orbital globe, `EPSG:4326` tactical map & live HUD |
| **Deployment** | Render (`https://meis.onrender.com/`) | Publicly hosted live mission control web interface |

```text
.
├── Archivo1.html           # Single-file 3D/2D tactical UI, responsive HUD & storm simulator
├── motor_astar.py          # A* tangent-ring router, Martian geodesy & EVA/PLSS physics model
├── servidor_api.py         # Lightweight HTTP API + static server (endpoints & CORS)
├── marte_telemetria.db     # SQLite database (sites, obstacles, atmosphere & 992 IAU features)
├── horizon_logo.jpg        # Boot splash brand asset
├── requirements.txt        # Runtime dependency manifest (Python 3 stdlib only)
├── .gitignore              # Clean exclusion rules for Python, web tooling, OS & secrets
└── README.md               # Project documentation
```

---

## 4. Local Installation & Quick Start

Because the backend is built purely on the Python 3 Standard Library, no external `pip` packages or database servers are needed.

### Prerequisites
- **Python 3.10+** (tested on **Python 3.13**)
- Internet access recommended for live NASA Mars Trek map tiles and CDN libraries (`Three.js`, `Globe.gl`, `Leaflet`), though backend routing and SQLite telemetry operate 100% locally.

### Setup & Run

1. **Clone the repository**
   ```bash
   git clone https://github.com/sierraivan14/MEIS-Mars-Environmental-Intelligence-System.git
   cd MEIS-Mars-Environmental-Intelligence-System
   ```

2. **(Optional) Create and verify virtual environment**
   ```bash
   python -m venv .venv
   # Windows:
   .venv\Scripts\activate
   # Linux / macOS:
   source .venv/bin/activate

   pip install -r requirements.txt
   ```

3. **Start the backend & web server**
   ```bash
   python servidor_api.py
   ```

4. **Open in browser**
   Navigate to **[http://localhost:8000](http://localhost:8000)**

### API Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/` | Serves the interactive 3D/2D mission interface (`Archivo1.html`) |
| `GET` | `/api/datos-mapa` | Returns SQLite sites, obstacles, atmospheric baselines, and `992` IAU features |
| `POST` | `/api/ruta-optima` | Computes optimal A\* path, EVA/PLSS consumption, and risk warnings (`origen`, `destino`, `tormenta_activa`) |
| `POST` | `/api/emergencia-tormenta` | Computes emergency A\* evacuation route to the nearest certified refuge (`nodo_actual`) |

---

## 5. Team Credits — `HORIZON.EXE` (NASA Space Apps Challenge 2026 — UNAQ Querétaro)

**Team:** `HORIZON.EXE`  
**Award:** *Galactic Problem Solver* — NASA Space Apps Challenge 2026

| Role | Member / Group | Key Contributions |
| :--- | :--- | :--- |
| **Backend Developer** | **[@sierraivan14](https://github.com/sierraivan14)** | Tactical **A\*** pathfinding engine (`ARES-PATH`), **SQLite** telemetry & schema design, **EVA/PLSS** metabolic/oxygen/power estimation formulas, REST API server, and **NASA/ESA** scientific dataset integration (**MOLA**, **HRSC**, **THEMIS**, IAU nomenclature). |
| **Frontend & Geospatial UI** | **Team `HORIZON.EXE`** | Interactive 3D orbital globe (`Three.js`/`Globe.gl`), 2D `EPSG:4326` Mars Trek map (`Leaflet`), zoom nomenclature, responsive HUD, and live dust-storm UI simulation. |
| **Planetary Research, Product & Pitch** | **Team `HORIZON.EXE`** | Planetary hazard research, regional survival metrics compilation, mission UX validation, and Space Apps jury presentation. |
