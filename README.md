# Map matching on OpenStreetMap, a practical tutorial

This tutorial is based on the [fast map matching program](https://github.com/cyang-kth/fmm).
It also supports road network in ESRI Shapefile.

### Demonstration

<img src="img/demo1.gif" width="400"/> <img src="img/demo2.gif" width="400"/>

### Content

- [1. Download routable network from OSM](#1-download-routable-network-from-osm)
- [2. Preprocess road network using PostGIS (can be skipped)](#2-preprocess-road-network-using-postgis-can-be-skipped)
    - [2.1 Import network into PostGIS database](#21-import-network-into-postgis-database)
    - [2.2 Complement bidirectional edges](#22-complement-bidirectional-edges)
    - [2.3 Export network as shapefile](#23-export-network-as-shapefile)
- [3. Run map matching with fmm](#3-run-map-matching-with-fmm)
    - [3.1 Run python](#31-run-python)
    - [3.2 Run web demo](#32-run-web-demo)

### Tools used

- [OSMNX](https://github.com/gboeing/osmnx) for downloading OSM data
- [Fast map matching program (Python extension)](https://github.com/cyang-kth/fmm) for map matching

### 1. Download routable network from OSM

[OSMNX](https://github.com/gboeing/osmnx) provides a convenient way to download a routable shapefile directly in python.

The original osmnx saves network in an undirected way, therefore we use the script below to save the graph to a bidirectional network where all edges are directed and bidirectional edges (osm tag one-way is false) are represented by two reverse edges.

<div align="center">
  <img src="img/dual.png" width="410"/>
</div>

```python
import osmnx as ox
import time
from shapely.geometry import Polygon
import os

def save_graph_shapefile_directional(G, filepath=None, encoding="utf-8"):
    # default filepath if none was provided
    if filepath is None:
        filepath = os.path.join(ox.settings.data_folder, "graph_shapefile")

    # if save folder does not already exist, create it (shapefiles
    # get saved as set of files)
    if not filepath == "" and not os.path.exists(filepath):
        os.makedirs(filepath)
    filepath_nodes = os.path.join(filepath, "nodes.shp")
    filepath_edges = os.path.join(filepath, "edges.shp")

    # convert undirected graph to gdfs and stringify non-numeric columns
    gdf_nodes, gdf_edges = ox.utils_graph.graph_to_gdfs(G)
    gdf_nodes = ox.io._stringify_nonnumeric_cols(gdf_nodes)
    gdf_edges = ox.io._stringify_nonnumeric_cols(gdf_edges)
    # We need an unique ID for each edge
    gdf_edges["fid"] = gdf_edges.index
    # save the nodes and edges as separate ESRI shapefiles
    gdf_nodes.to_file(filepath_nodes, encoding=encoding)
    gdf_edges.to_file(filepath_edges, encoding=encoding)

print("osmnx version",ox.__version__)

# Download by a bounding box
bounds = (17.4110711999999985,18.4494298999999984,59.1412578999999994,59.8280297000000019)
x1,x2,y1,y2 = bounds
boundary_polygon = Polygon([(x1,y1),(x2,y1),(x2,y2),(x1,y2)])
G = ox.graph_from_polygon(boundary_polygon, network_type='drive')
start_time = time.time()
save_graph_shapefile_directional(G, filepath='./network-new')
print("--- %s seconds ---" % (time.time() - start_time))

# Download by place name
place ="Stockholm, Sweden"
G = ox.graph_from_place(place, network_type='drive', which_result=2)
save_graph_shapefile_directional(G, filepath='stockholm')

# Download by a boundary polygon in geojson
import osmnx as ox
from shapely.geometry import shape
json_file = open("stockholm_boundary.geojson")
import json
data = json.load(json_file)
boundary_polygon = shape(data["features"][0]['geometry'])
G = ox.graph_from_polygon(boundary_polygon, network_type='drive')
save_graph_shapefile_directional(G, filepath='stockholm')
```

In the third manner, here is a screenshot of the network in QGIS.

<div align="center">
  <img src="img/raw_data.png" width="410"/>
</div>

### 2. Preprocess road network using PostGIS (can be skipped)

**Since the complementation is already done in Python using osmnx, there is no need to preprocess the network in PostGIS. You can directly go to [step 3](#3-run-map-matching-with-fmm).**

Although the network shapefile contains topology information (from, to), these information cannot be directly used in fmm. We need to preprocess the data
1. **relabel the `from`, `to` attribute from the original OSM id to smaller integers**. The original from/to node IDs generated by OSMNX are quite big integers which will cause numeric problems in the map matching program. Therefore, the relabel is needed.
2. Duplicate bidirectional edges, i.e., add a reverse edge if an edge's `oneway` equals `False`.  

The two tasks can be done in PostGIS using the following code

#### 2.1 Import network into PostGIS database

Before you import the shapefile into PostGIS database. You need to create a new database or use an existing one.
Following the [instructions](http://www.postgresqltutorial.com/postgresql-create-database/). It is highly recommended to use PgAdmin3.

Assume that the database name is called `mm_db` by user `postgres`. (You may replace the user name `postgres` and database name `mm_db` with your own settings.)

You need to first have `postgis` extension installed for your database. Run the following code in **bash shell**.

```
psql -U postgres -d mm_db -c "
CREATE EXTENSION postgis IF NOT EXISTS;
CREATE EXTENSION postgis_topology IF NOT EXISTS"
```

We use shp2pgsql for importing the shapefile into `mm_db`. Run the following code in **bash shell**.

```
# Create a schema called network
psql -U postgres -d mm_db -c "CREATE SCHEMA network;"
# Import the downloaded edges and nodes shapefile.
shp2pgsql data/stockholm/edges/edges.shp network.original_edges | psql -U postgres -d mm_db
shp2pgsql data/stockholm/nodes/nodes.shp network.original_nodes | psql -U postgres -d mm_db
```

#### 2.2 Complement bidirectional edges

**Execute the following commands in `psql` or PgAdmin**. Type `psql -U postgres -d mm_db` in bash shell to open the psql.

```
-- Add two columns in the original network to store the new IDs.

ALTER TABLE network.original_edges ADD COLUMN source int;
ALTER TABLE network.original_edges ADD COLUMN target int;

-- Update the columns by joining with the original_nodes table

UPDATE network.original_edges AS a
SET source = b.gid
FROM network.original_nodes AS b
WHERE a.from = b.osmid;

UPDATE network.original_edges AS a
SET target = b.gid
FROM network.original_nodes AS b
WHERE a.to = b.osmid;

-- Create a new table to include the dual edges
CREATE TABLE network.dual
(
  gid serial NOT NULL,
  geom geometry(LineString),
  source integer,
  target integer,
  CONSTRAINT dual_pkey PRIMARY KEY (gid)
);

-- Insert original edges

INSERT INTO
    network.dual (geom,source,target)
SELECT ST_LineMerge(geom),source,target FROM network.original_edges
WHERE ST_GeometryType(ST_LineMerge(geom))='ST_LineString';

-- Insert reverse edges whose oneway equals false

INSERT INTO
    network.dual (geom,source,target)
SELECT ST_Reverse(ST_LineMerge(geom)),target,source FROM network.original_edges
WHERE oneway='False' AND ST_GeometryType(ST_LineMerge(geom))='ST_LineString';
```

#### 2.3 Export network as shapefile

Run the code in **bash shell**

```
pgsql2shp -f data/stockholm/network_dual.shp -h localhost -u postgres -d mm_db "SELECT gid::integer as id,source,target,geom from network.dual"
```

You can see the difference between the original network (left) and the new network (right).

<img src="img/single.png" width="400"/> <img src="img/dual.png" width="410"/>

### 3. Run map matching with fmm

Install the [**fmm** program](https://github.com/cyang-kth/fmm) in **C++ and Python** extension following the [instructions](https://github.com/cyang-kth/fmm/wiki).

#### 3.1 Run python

```python
from fmm import FastMapMatch,Network,NetworkGraph,UBODTGenAlgorithm,UBODT,FastMapMatchConfig

# Read network data
# The default column names are id, source, target. Update the values in your case
network = Network("network/edges.shp","fid","u","v")
print "Nodes {} edges {}".format(network.get_node_count(),network.get_edge_count())
graph = NetworkGraph(network)

# Can be skipped if you already generated an ubodt file
ubodt_gen = UBODTGenAlgorithm(network,graph)
status = ubodt_gen.generate_ubodt("network/ubodt.txt", 0.02, binary=False, use_omp=True)
print status

# Read UBODT

ubodt = UBODT.read_ubodt_csv("network/ubodt.txt")
model = FastMapMatch(network,graph,ubodt)

k = 8
radius = 0.003
gps_error = 0.0005
fmm_config = FastMapMatchConfig(k,radius,gps_error)

# Run map matching for wkt

wkt = "LineString(104.10348 30.71363,104.10348 30.71363,104.10348 30.71363)"
result = model.match_wkt(wkt,fmm_config)
```

Check more examples at https://github.com/cyang-kth/fmm/blob/master/example/notebook/fmm_example.ipynb.


#### 3.2 Run web demo


The following python web library are needed.

```
pip install tornado flask
```

After the Python extension of fmm is installed, run the [fmm_web_app.py](https://github.com/cyang-kth/fmm/tree/master/example/web_demo) provided in the repo of
`fmm` using the provided json configuration file [fmm_config.json](fmm_config.json).
or [stmatch_config.json](stmatch_config.json)

```
python FMM_DIR/example/web_demo/fmm_web_app.py -c fmm_config.json
```

```
python FMM_DIR/example/web_demo/fmm_web_app.py -c stmatch_config.json
```

Visit [http://localhost:5000/demo](http://localhost:5000/demo) to open the drawing tools where you can draw a trajectory and it will be matched to the OSM, as shown below.

<img src="img/demo3.gif" width="400"/> <img src="img/demo4.gif" width="400"/>


### Contact information

Can Yang, Ph.D. student at KTH, Royal Institute of Technology in Sweden

Email: cyang(at)kth.se

Homepage: https://people.kth.se/~cyang/
