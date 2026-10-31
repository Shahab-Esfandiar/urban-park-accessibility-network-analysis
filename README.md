# Evaluating Urban Green Space Equity: Network-Based Accessibility & Demographic Catchment Modeling

An applied Spatial Data Science case study evaluating pedestrian accessibility, demographic pressure, and spatial equity across public parks in the Çukurova district (Adana, Turkey). By bridging graph theory (OSMnx/NetworkX) with high-resolution demographic grids (WorldPop 10m), this project uncovers the spatial fallacies of conventional circular buffers in municipal planning.

[![Python](https://img.shields.io/badge/Python-3.9%2B-blue.svg)](https://www.python.org/)
[![GeoPandas](https://img.shields.io/badge/GeoPandas-Spatial%20Dataframe-green.svg)](https://geopandas.org/)
[![NetworkX](https://img.shields.io/badge/NetworkX-Graph%20Analysis-orange.svg)](https://networkx.org/)
[![OSMnx](https://img.shields.io/badge/OSMnx-Pedestrian%20Network-purple.svg)](https://osmnx.readthedocs.io/)

---

### Project Snapshot & Key Metrics

| Metric | Measured Value | Standard / Benchmark | Status |
| :--- | :---: | :---: | :---: |
| **Study Area Footprint** | 4,283 ha | Çukurova Metropolitan District | Municipal boundary |
| **Residential Fabric Coverage** | 79% of total land use | — | Predominantly residential |
| **Park Surface Allocation** | < 3% | — | Severe land deficit |
| **Green Space per Capita** | **1.8 m² / person** | 10.0 m² / person (National Standard) | **-82% Deficit** |
| **Audited Local Parks** | 155 parks | OpenStreetMap & Municipal Datasets | Complete district audit |
| **Overall Accessible Population** | 64.16% | Walking threshold ≤ 500 m | Realistic network access |
| **Excluded / Underserved Population**| **35.84%** | Over 120,000 residents | Out of service radius |
| **High-Priority Deficit Sites** | 69 locations | PDI_norm < 5% | Immediate intervention |

---

## 1. Spatial Planning Dilemma

Urban densification across Çukurova has introduced critical spatial disparities between population concentration and public green amenities:

* **Overestimated Coverage:** Municipalities often rely on straight-line "as-the-crow-flies" (Euclidean) distances, falsely reporting adequate park coverage while ignoring barriers like cul-de-sacs, major arterial corridors, and walled superblocks.
* **Demographic Bottlenecks:** A park with a nominal 500-meter service radius may technically cover thousands of citizens, but high population density leads to severe congestion and rapid amenity degradation.
* **Objective:** This study models true walking paths using graph shortest-path routing (Dijkstra) integrated with a 10-meter resolution population grid ($373,468$ inhabited data points) to pinpoint service deserts and optimize future park allocations.

---

## 2. Spatial Data & Preprocessing Pipeline

To eliminate distortion in metric distance and polygon surface calculations, all spatial layers were projected to **UTM Zone 36N (EPSG:32636)**:

* **Urban Park Polygons ($n = 155$):** Extracted from OSM and municipal GIS records, cleaned to remove topological slivers, and calculated for exact metric footprint area.
* **Pedestrian Walkability Graph:** Queried via OSMnx (`network_type="walk"`), projected and converted to a planar NetworkX undirected graph where edge weights reflect geometric length in meters.
* **Disaggregated Population Grid:** 10m population density raster translated into point vector features via affine transformation. Inhabited cells ($>0$ density) were isolated to model local population loads.
* **Residential Parcels:** Filtered from primary land use shapefiles to establish target demand zones against available park supplies.

---

## 3. Analytical Framework & Mathematical Formulation

The modeling workflow links network topology with demographic overlay:

```text
Park Geometries (EPSG:32636)              Pedestrian Walk Network (OSM)
            │                                           │
            └───────────────┐   ┌───────────────────────┘
                            ▼   ▼
               [Boundary Node Snapping]
               Find nearest network nodes: u = argmin ||v_k - u||
                            │
                            ▼
               [Dijkstra Reachability Analysis]
               Extract reachable nodes within walking threshold: d_G(u, v) ≤ 500 m
                            │
                            ▼
               [Network Catchment Polygon (S_net)]
               ConvexHull(Nodes) \ Park_Geometry
                            │
                            ▼
               [10m Population Density Overlay]
               Aggregate population points inside catchment: Pop = Σ ρ_j
                            │
                            ▼
               [Population Density Index (PDI)]
               Quantify human pressure per m² of accessible green space
                            │
                            ▼
               [Intervention Site Allocation]
               Flag critical catchments (PDI_norm < 5%) ──► 69 new park centroids
```

### 3.1 Network-Constrained Catchment
For each park polygon $P_i$ with perimeter vertices $V_i$, accessible nodes $\mathcal{N}_{500}$ on the pedestrian graph $G = (V, E, W)$ are extracted via Dijkstra's shortest path:

$$d_G(u, v) \le 500 \quad \text{meters}, \quad \forall u \in \text{NearestNodes}(V_i)$$

The actual walkable service footprint $S_i^{\text{net}}$ is defined by the outer convex hull of accessible nodes excluding the park surface itself:

$$S_i^{\text{net}} = \text{ConvexHull}(\mathcal{N}_{500}) \setminus P_i$$

### 3.2 Population Density Index (PDI)
To evaluate congestion and demographic load, population points $\rho_j$ within each service polygon are aggregated:

$$\text{Pop}_i = \sum_{j \in S_i^{\text{net}}} \rho_j$$

The raw Population Density Index ($PDI_i$) measures the population pressure supported per square meter of catchment area:

$$PDI_i = \frac{\text{Pop}_i}{\text{Area}(S_i^{\text{net}})}$$

For district-wide benchmarking, values are min-max normalized to a 0–100% relative score:

$$PDI_{i, \text{norm}} = \left( \frac{PDI_i - PDI_{\min}}{PDI_{\max} - PDI_{\min}} \right) \times 100$$

---

## 4. Empirical Evaluation & Key Findings

### 4.1 Euclidean vs. Network Routing Disparity
Euclidean buffers overestimate actual walking catchments by **28% to 42%**, generating a false sense of accessibility in municipal master plans:

<div align="center">
  <img width="1255" height="1924" alt="Comparison of Euclidean and Network Buffers" src="https://github.com/user-attachments/assets/b131e051-925b-4d53-93fe-d0960fb5cba2" />
  <p><em>Figure 1: Visual comparison between circular 500m Euclidean buffers (blue) and pedestrian network service areas (red) across sample urban parks.</em></p>
</div>

* **Urban Block Detours:** Residents living within 500 meters straight-line distance frequently walk **750 to 900 meters** on street paths due to closed communities and missing pedestrian links.
* **Catchment Distortion:** As seen in Figure 1, network buffers conform strictly to street layout, revealing unserved urban pockets directly adjacent to parks.

### 4.2 Catchment Evaluation Matrix (Sample Parks)

The table below highlights the stark divergence between physical park area, surrounding population load, and normalized accessibility scores:

| Park ID | Footprint Area (m²) | Catchment Pop. (10m grid) | Raw PDI (cap/m²) | Normalized PDI (%) | Minimum Size Standard (≥ 24,865 m²) |
| :---: | :---: | :---: | :---: | :---: | :---: |
| **Park 0** | 459,159.74 | 5,777,150 | 12.58 | 2.64% | Compliant |
| **Park 1** | 678,482.08 | 9,154,312 | 13.49 | 3.02% | Compliant |
| **Park 2** | 595,456.36 | 9,007,954 | 15.13 | 3.70% | Compliant |
| **Park 3** | 140,217.65 | 3,123,861 | 22.28 | 6.70% | Compliant |
| **Park 4** | 202,421.20 | 7,131,829 | 35.23 | 12.12% | Compliant |
| **Park 152** | 398,664.97 | 2,504,004 | 6.28 | **0.00% (Lowest)** | Compliant |

---

## 5. Spatial Site Selection: 69 Proposed Park Allocations

To resolve the spatial accessibility deficit, a spatial optimization heuristic was deployed across the district:

1. **Deficit Identification:** Catchments exhibiting extreme demographic pressure combined with low service scores ($PDI_{\text{norm}} < 5\%$) were classified as underserved.
2. **Centroid Allocation:** Geometric centroids of underserved residential zones were isolated to serve as candidate intervention anchors.
3. **Service Footprint:** Each proposed location was modeled with a localized **250-meter walking buffer** to bridge local accessibility voids without requiring extensive land acquisition.

<div align="center">
  <img width="1279" height="711" alt="Proposed Park Locations Map" src="https://github.com/user-attachments/assets/17a60a1d-0030-45d8-9d14-54771319cff1" />
  <p><em>Figure 2: Spatial distribution of existing urban parks alongside 69 newly proposed park allocations overlaid on the Çukurova road network.</em></p>
</div>

---

## 6. Actionable Urban Planning Recommendations

1. **Pocket Parks in High-Density Quarters:** Highly populated sectors (such as Mahfesığmaz) lack large vacant parcels. The municipality should convert underutilized urban parcels into pocket parks ($250\text{ m}$ catchment) to immediately relieve local pressure.
2. **Pedestrian Permeability Upgrades:** Installing mid-block pedestrian connections, traffic-calmed alleys, and crosswalks expands the reach of existing parks by up to 25% without acquiring new real estate.
3. **Multi-Neighborhood Boundary Strategy:** Large parks situated near neighborhood boundaries (e.g., Karslılar, Toros) serve broader multi-district catchments and should receive prioritized budget for transit connectivity and lighting.

---

## 7. Toolchain & Dependencies

```text
Spatial Frameworks  : GeoPandas, Shapely, Contextily
Network Analytics   : OSMnx, NetworkX
Raster Engine       : Rasterio, NumPy
Data Visualization  : Matplotlib, Pandas, Tqdm
Coordinate System   : EPSG:32636 (WGS 84 / UTM Zone 36N)
```
