# ODI Building Footprints POC

A proof-of-concept demonstrating a POC of California ODI's building footprints data pipeline (AWS MWAA + Snowflake + dbt) in **Azure Databricks**.

## Overview

This project processes Microsoft's Global ML Building Footprints for California, joins them with US Census TIGER/Line boundaries, and publishes the results in GeoParquet and Shapefile formats.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Databricks Job                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────┐    ┌──────────────────────┐                       │
│  │ Load MS Building     │    │ Load Census TIGER    │                       │
│  │ Footprints (CA)      │    │ Line Shapefiles      │                       │
│  └──────────┬───────────┘    └──────────┬───────────┘                       │
│             │                           │                                   │
│             │         Bronze Layer      │                                   │
│             └───────────┬───────────────┘                                   │
│                         │                                                   │
│                         ▼                                                   │
│             ┌───────────────────────┐                                       │
│             │ Spatial Join          │                                       │
│             │ Buildings → Blocks    │                                       │
│             │ Buildings → Places    │                                       │
│             └───────────┬───────────┘                                       │
│                         │                                                   │
│                         │         Silver Layer                              │
│                         ▼                                                   │
│             ┌───────────────────────┐                                       │
│             │ Publish to            │                                       │
│             │ GeoParquet & Shapefile│                                       │
│             └───────────────────────┘                                       │
│                         │                                                   │
│                         │         Gold Layer                                │
│                         ▼                                                   │
│             ┌───────────────────────┐                                       │
│             │ Unity Catalog Volume  │                                       │
│             │ /Volumes/.../parquet/ │                                       │
│             │ /Volumes/.../shp/     │                                       │
│             └───────────────────────┘                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
├── README.md
├── LICENSE
├── .gitignore
├── jobs/
│   └── odi_building_footprints.yaml       # Databricks workflow definition
└── notebooks/
    ├── create_odi_storage_location_and_schema.ipynb   # Prerequisite: Unity Catalog setup
    ├── load_global_ml_building_footprints_ca.ipynb    # Load MS Building Footprints
    ├── load_census_tiger_line_shape_files.ipynb       # Load TIGER/Line data
    ├── join_odi_building_footprints_with_census_blocks.ipynb  # Spatial joins
    └── publish_odi_footprints_with_census_blocks_data.ipynb   # Export outputs
```

## Data Sources

| Source | Description | URL |
|--------|-------------|-----|
| Microsoft Building Footprints | ML-derived building polygons | [Global ML Building Footprints](https://github.com/microsoft/GlobalMLBuildingFootprints) |
| US Census TIGER/Line | Census boundaries (blocks, places, counties, tracts) | [TIGER/Line Shapefiles](https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html) |

## Pipeline Details

### Task 1: Load Microsoft Building Footprints

**Notebook**: `load_global_ml_building_footprints_ca.ipynb`

- Reads the dataset index from Microsoft's CDN
- Identifies quadkeys that intersect California using `mercantile`
- Downloads GeoJSONSeq files for each quadkey
- Writes to Delta table: `{catalog}.{schema}.global_ml_building_footprints_california`

**Key Libraries**: `geopandas`, `mercantile`, `shapely`, `fsspec`

### Task 2: Load Census TIGER/Line Shapefiles

**Notebook**: `load_census_tiger_line_shape_files.ipynb`

- Uses `pygris` library to download TIGER/Line data
- Loads state-specific data: counties, tracts, block_groups, blocks, places, pumas, roads
- Loads US-wide data: coastline, nation, states, urban_areas, etc.
- Writes to Delta tables with `tiger_` prefix

**Key Libraries**: `pygris`, `geopandas`

### Task 3: Spatial Join

**Notebook**: `join_odi_building_footprints_with_census_blocks.ipynb`

- Joins buildings to Census blocks using `ST_Intersects`
- Deduplicates by selecting block with largest intersection area
- Joins to Census places (cities/towns) with same logic
- Calculates building area in square meters
- Writes to: `{catalog}.odi_silver.building_footprints_with_blocks_and_places_{year}`

**Key Functions**: `st_intersects`, `st_intersection`, `st_area`, `st_buffer`

### Task 4: Publish Outputs

**Notebook**: `publish_odi_footprints_with_census_blocks_data.ipynb`

- Exports data partitioned by county
- **GeoParquet**: Efficient columnar format for analytics
- **Zipped Shapefile**: Compatible with Esri and traditional GIS tools
- Writes to Unity Catalog Volume

## Workflow Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `catalog` | `odi_datalake` | Unity Catalog name |
| `schema_name` | `odi_bronze` | Schema for raw data |
| `state` | `CA` | US state abbreviation |
| `tiger_year` | `2020` | Census TIGER year |
| `volume_path` | `/Volumes/odi_datalake/odi_gold/global_ml_building_footprints` | Output volume |
| `source_table` | `odi_datalake.odi_silver.building_footprints_with_blocks_and_places_` | Source for publishing |
| `catalog_storage_location` | `odi_datalake` | Storage location for catalog |
| `schema_storage_location` | `odi_datalake` | Storage location for schema |

## Prerequisites

### Azure Databricks Workspace

- Unity Catalog enabled
- Serverless compute or cluster with:
  - Runtime 14.3 LTS or higher
  - Access to install PyPI packages
- Refer to this public repo as an example of basic Azure Databricks workspace creation through infrastructure as code (IAC)
   - [azure_databricks_basic_private_connectivity](https://github.com/benkarabinus/azure_databricks_basic_private_connectivity)

### Unity Catalog Setup

> **Run `create_odi_storage_location_and_schema.ipynb` once before executing the Databricks workflow for the first time.** This notebook is not part of the job — it is a one-time manual setup step.

The **`create_odi_storage_location_and_schema.ipynb`** notebook automates the full Unity Catalog setup in one pass:

- Creates the `odi_datalake` Unity Catalog, optionally backed by an external storage location (recommended) or the workspace default metastore (suitable for POC)
- Creates the three medallion schemas: `odi_bronze`, `odi_silver`, `odi_gold`
- Creates the `supporting_geometry_files` volume in `odi_bronze` (used for state geometry reference files)
- Creates the `global_ml_building_footprints` volume in `odi_gold` (output destination)
- Includes a verification step to confirm all resources were created successfully

After running the notebook, upload the California geometry file to:
```
/Volumes/odi_datalake/odi_bronze/supporting_geometry_files/state_geometries/california.geojson
```


## Running the Pipeline

### Option 1: Databricks Git Folders (recommended starting point)

> **Docs**: [Git integration for Databricks Git Folders](https://learn.microsoft.com/en-us/azure/databricks/repos/)

This is the simplest approach — link your Databricks workspace directly to GitHub.

#### Setup

1. In Databricks, go to **Workspace** → your user folder
2. Click **Create** → **Git folder**
3. Enter the repository URL and authenticate
4. The notebooks sync automatically

#### Workflow

```
Edit locally → Push to GitHub → Pull in Databricks → Run job
```

To sync changes:
1. Open the Git folder in Databricks
2. Click **Pull** to get the latest changes
3. The job automatically uses the updated notebooks

#### Creating the Job

1. Go to **Workflows** → **Create Job**
2. Add tasks pointing to notebooks in your Git folder
3. Configure parameters and dependencies
4. The job definition is saved in `jobs/odi_building_footprints.yaml` for reference

### Option 2: GitHub Actions CI/CD (Alternative)

> **Docs**: [CI/CD with GitHub Actions on Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/ci-cd/ci-cd-github-actions)

For environments without Databricks Git Folders, or when you want automated deployment on every push.

> **Note**: Requires GitHub-hosted runners or a self-hosted runner to be available.

#### Setup

1. **Create a Databricks Personal Access Token**:
   - Go to your Databricks workspace
   - Click your profile (top right) → **Settings**
   - Select **Developer** → **Access tokens**
   - Click **Generate new token** and copy the value

2. **Add GitHub Secrets** (Settings → Secrets and variables → Actions):

   | Secret | Value |
   |--------|-------|
   | `DATABRICKS_HOST` | Your workspace URL (e.g., `https://adb-xxx.azuredatabricks.net`) |
   | `DATABRICKS_TOKEN` | The token from step 1 |

3. **Trigger deployment**:
   - **Automatic**: Push to `main` branch
   - **Manual**: Go to Actions → "Deploy to Databricks" → Run workflow

#### What the Workflow Does

| Trigger | Action |
|---------|--------|
| Pull Request | Validate files only (no deploy) |
| Push to main | Deploy notebooks + create/update job |
| Manual dispatch | Deploy + optionally run the job |

### Option 3: Databricks CLI (Manual)

> **Docs**: [Databricks CLI](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/cli/)

```bash
# Install Databricks CLI
pip install databricks-cli

# Configure authentication
databricks configure --token

# Upload notebooks
databricks workspace import-dir ./notebooks /Workspace/Shared/odi_building_footprints/notebooks

# Create job (first time)
databricks jobs create --json-file job_config.json
```

## Output Tables

### Bronze Layer

| Table | Description |
|-------|-------------|
| `global_ml_building_footprints_california` | Raw building footprints with geometry_wkt |
| `tiger_blocks` | Census block boundaries |
| `tiger_places` | Census places (cities, towns) |
| `tiger_counties` | County boundaries |
| `tiger_tracts` | Census tracts |
| ... | Additional TIGER datasets |

### Silver Layer

| Table | Description |
|-------|-------------|
| `building_footprints_with_blocks_and_places_{year}` | Buildings joined to Census geography |

### Gold Layer (Volume)

| Path | Format | Description |
|------|--------|-------------|
| `/Volumes/.../parquet/{year}/county_fips_*.parquet` | GeoParquet | By-county exports |
| `/Volumes/.../shp/{year}/county_fips_*.zip` | Shapefile | By-county exports |

## Comparison with Original ODI Implementation

| Aspect | ODI (AWS/Snowflake/dbt) | This POC (Azure Databricks) |
|--------|-------------------------|----------------------------|
| **Orchestration** | AWS MWAA (Airflow) | Databricks Workflows |
| **Data Warehouse** | Snowflake | Delta Lake + Unity Catalog |
| **Transformations** | dbt SQL models | Databricks Notebooks (Python/SQL) |
| **Geospatial** | Snowflake ST functions | Databricks ST functions |
| **CI/CD** | dbt Cloud / GitHub Actions | Databricks CLI + GitHub Actions |
| **Output Formats** | GeoParquet, Shapefile | GeoParquet, Shapefile |

## License

MIT License - See [LICENSE](LICENSE)
