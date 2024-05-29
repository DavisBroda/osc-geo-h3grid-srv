# End to End Examples

## Common Setup

Before any of the examples below are run it is assumed that the below 
setup operations will have been performed. 

### Setting up your Environment

Some environment variables are used by the examples.
Set up these environment variables as follows (note that `source` is used)
~~~~
source ./bin/environment.sh
~~~~

It is recommended that a Python virtual environment be created.
Several convenience scripts are available to create and activate
a virtual environment.

To create a new virtual environment run the below command
(it will create a directory called "venv" in your current working directory):
~~~~
$PROJECT_DIR/bin/venv.sh
~~~~

Once your virtual environment has been created, it can be activated
as follows (note: you *must* activate the virtual environment
for it to be used, and the command requires `source` to ensure
environment variables to support venv are established correctly):
~~~~
source $PROJECT_DIR/bin/vactivate.sh
~~~~

Install the required libraries as follows:
~~~~
pip install -r requirements.txt
~~~~

To run command line commands the geoserver package must first be installed
by pip. Use the below commands to install into venv.

~~~
cd ./src
pip install -e .
~~~

### Configuring the CLI

A CLI is available that makes it easy to interact
with the Geoserver:

The CLI communicates with a server (Sqlite-based or ETCD-based)
and hence requires a host and port. For your convenience,
this tutorial uses HOST and PORT. 

Also, verbose logging can be enabled using the "--verbose" tag.
For your convenience, each of the examples in this tutorial
use an environment variable, VERBOSE, which if set to
"--verbose" will permit extended logging in the CLI.

Local host is set up to use port 8001:
~~~~
HOST=localhost ;
PORT=8000 ;
VERBOSE="--verbose"
~~~~

Docker host is set up to use port 25001:
~~~~
HOST=localhost ;
PORT=24000 ;
VERBOSE="--verbose"
~~~~

To disable verbose logging, unset VERBOSE:
~~~~
VERBOSE=""
~~~~


## Belgian Flood Data as Interpolated H3

This example will take historical flood data from europe 
(specifically the 10-year flood) and will convert it from its original format
(TIFF) into a format the loaders can process (parquet). It 
will then interpolate and load this dataset, and generate a 
visualization of it. 

### Retrieving Data & Shapefiles

Shapefiles are files that define a geographic region. They are used in this
example to ensure that processing only happens within a target region. 
In order to run the below examples, shapefiles will need to be downloaded from
the following link (if not already downloaded from the GISS Temperature example 
in the getting-started README):

Shapefiles source:
- [world-administrative-boundaries.zip](https://public.opendatasoft.com/api/explore/v2.1/catalog/datasets/world-administrative-boundaries/exports/shp?lang=en&timezone=America%2FNew_York): 

Retrieved from parent site: https://public.opendatasoft.com/explore/dataset/world-administrative-boundaries/export/
- retrieved as a dataset from the "Geographic file formats" section, 
"Shapefile" element, by clicking the "Whole dataset" link

Create the `data/shapefiles/WORLD` directory as below (if it does not already exist)
~~~
mkdir -p ./data/shapefiles/WORLD
~~~

Unzip the `world-administrative-boundaries.zip` file into the
`data/shapefiles/WORLD` directory. This should result in a 
directory structure that looks like below:

~~~
data
|-- shapefiles
    |-- WORLD
        |-- world-adminstrative-boundaries.prj
        |-- world-adminstrative-boundaries.cpg
        |-- world-adminstrative-boundaries.dbf
        |-- world-adminstrative-boundaries.shp
        |-- world-adminstrative-boundaries.shx
~~~


Additionally, the flood data that will be used as the 
raw data for this example will need to be retrieved. Note that this
data is 5GB in size.

It can be retrieved from the below link
- [Pan-European data sets of river flood probability of occurrence under present and future climate_1_all.zip](https://data.4tu.nl/file/df7b63b0-1114-4515-a562-117ca165dc5b/5e6e4334-15b5-4721-a88d-0c8ca34aee17)

Which was retrieved from this [parent site](https://data.4tu.nl/articles/dataset/Pan-European_data_sets_of_river_flood_probability_of_occurrence_under_present_and_future_climate/12708122) 

Create the `data/geo_data/flood/europe_flood_data` directory as below:

~~~
mkdir -p ./data/geo_data/flood/europe_flood_data
~~~

Unzip the `Pan-European data sets of river flood probability of occurrence under present and future climate_1_all.zip`
file into the `data/geo_data/flood/europe_flood_data` directory.
This should result in a directory structure that looks like the below:

~~~
data
|-- geo_data
    |-- flood
        |-- europe_flood_data
            |-- data.zip
            |-- readme_river_floods_v1.1.pdf
~~~

Create the `data/geo_data/flood/europe_flood_data/data` directory as below

~~~
mkdir -p ./data/geo_data/flood/europe_flood_data/data
~~~

Unzip the `data.zip` file into the `./data/geo_data/flood/europe_flood_data/data`
directory. This should result in a file structure like below:

~~~
data
|-- geo_data
    |-- flood
        |-- europe_flood_data
            |-- data.zip
            |-- readme_river_floods_v1.1.pdf
            |-- data
                |-- River_discharge_1971_2000_hist.dbf
                |-- River_discharge_1971_2000_hist.prj
                ...
~~~

### Convert Tiff to Parquet

There is no loader for tiff files, so the tiff file is first converted to
a parquet file to allow for loading. This will create the output file
`./tmp/flood_depth_10_year_belgium.parquet`.

```
RAW="./data/geo_data/flood/europe_flood_data/data/River_flood_depth_1971_2000_hist_0010y.tif" ; 
OUT="./tmp/flood_depth_10_year_belgium.parquet" ;
FILTER="Belgium" ;

python ./examples/loading/flood_data/flood_to_parquet.py \
--raw $RAW \
--output $OUT \
--filter $FILTER
```

### Load & Interpolate

The data will be loaded and interpolated as an h3 grid of various resolutions,
up to the maximum specified in the configuration file. This will create
the `./tmp/flood_depth_10_year_belgium.duckdb` file as output. This command
may take several minutes to execute.

```
CONFIG_PATH="./examples/loading/flood_data/flood_depth_10_year_belgium.yml" ;

python ./src/geoserver/cli_load.py --host $HOST --port $PORT load \
--config_path $CONFIG_PATH 
```

### Add metadata entry

In order to interact with a dataset its metadata must be registered.
The below command will register the dataset created in previous steps.

If no metadata database existed previously, this will create the 
`./tmp/dataset_metadata.duckdb` file.

~~~
DATABASE_DIR="./tmp" ;
DATASET_NAME="flood_depth_10_year_belgium" ;
DESCRIPTION="Flood depth in belgium during 10 year flood" ;
VALUE_COLUMNS="{\"value\":\"REAL\"}" ;
INTERVAL="one_time" ;
DATASET_TYPE="h3" ;
python ./src/geoserver/cli_geospatial.py $VERBOSE --host $HOST --port $PORT addmeta \
    --database_dir $DATABASE_DIR \
    --dataset_name $DATASET_NAME \
    --description "$DESCRIPTION" \
    --value_columns $VALUE_COLUMNS \
    --interval $INTERVAL \
    --dataset_type $DATASET_TYPE	
~~~


### Visualize Dataset

This will create a visualization of the data in the interpolated
dataset created in prior steps. It will generate the visualization
based on the resolution 7 h3 grid (hexagons with area of 
approximately 5 km^2), and will use a blue colour scale. 

This will create an output file at `./tmp/flood_depth_10_year_belgium.html`
contianing the visualization. This file can be viewed with any internet browser. 

~~~
DATABASE_DIR="./tmp" ;
DATASET="flood_depth_10_year_belgium" ;
RESOLUTION=7 ;
VALUE_COLUMN="value" ;
RED=0 ;
GREEN=0 ;
BLUE=255 ;
OUTPUT_FILE="./tmp/flood_depth_10_year_belgium.html" ;
MIN_LAT=49.25 ;
MAX_LAT=51.55 ;
MIN_LONG=2.19 ;
MAX_LONG=6.62 ;

python ./src/geoserver/cli_geospatial.py $VERBOSE --host $HOST --port $PORT visualize-dataset \
--database-dir $DATABASE_DIR \
--dataset $DATASET \
--resolution $RESOLUTION \
--value-column $VALUE_COLUMN \
--max-color $RED $GREEN $BLUE \
--output-file $OUTPUT_FILE \
--min-lat $MIN_LAT \
--max-lat $MAX_LAT \
--min-long $MIN_LONG \
--max-long $MAX_LONG 
~~~

## Batch processing of flood data

This example provides a script that will automatically iterate through 
many datasets and generate visualizations of them. Note that the script that
does this will take a very long time (potentially several days) to run to completion.

### Retrieving Data & Shapefiles

The shapefiles and data used in this example are the same as in the Belgium
example above, so if that example has already been run, this section can be skipped


Shapefiles are files that define a geographic region. They are used in this
example to ensure that processing only happens within a target region. 
In order to run the below examples, shapefiles will need to be downloaded from
the following link (if not already downloaded from the GISS Temperature example 
in the getting-started README):

Shapefiles source:
- [world-administrative-boundaries.zip](https://public.opendatasoft.com/api/explore/v2.1/catalog/datasets/world-administrative-boundaries/exports/shp?lang=en&timezone=America%2FNew_York): 

Retrieved from parent site: https://public.opendatasoft.com/explore/dataset/world-administrative-boundaries/export/
- retrieved as a dataset from the "Geographic file formats" section, 
"Shapefile" element, by clicking the "Whole dataset" link

Create the `data/shapefiles/WORLD` directory as below (if it does not already exist)
~~~
mkdir -p ./data/shapefiles/WORLD
~~~

Unzip the `world-administrative-boundaries.zip` file into the
`data/shapefiles/WORLD` directory. This should result in a 
directory structure that looks like below:

~~~
data
|-- shapefiles
    |-- WORLD
        |-- world-adminstrative-boundaries.prj
        |-- world-adminstrative-boundaries.cpg
        |-- world-adminstrative-boundaries.dbf
        |-- world-adminstrative-boundaries.shp
        |-- world-adminstrative-boundaries.shx
~~~


Additionally, the flood data that will be used as the 
raw data for this example will need to be retrieved. Note that this
data is 5GB in size.

It can be retrieved from the below link
- [Pan-European data sets of river flood probability of occurrence under present and future climate_1_all.zip](https://data.4tu.nl/file/df7b63b0-1114-4515-a562-117ca165dc5b/5e6e4334-15b5-4721-a88d-0c8ca34aee17)

Which was retrieved from this [parent site](https://data.4tu.nl/articles/dataset/Pan-European_data_sets_of_river_flood_probability_of_occurrence_under_present_and_future_climate/12708122) 

Create the `data/geo_data/flood/europe_flood_data` directory as below:

~~~
mkdir -p ./data/geo_data/flood/europe_flood_data
~~~

Unzip the `Pan-European data sets of river flood probability of occurrence under present and future climate_1_all.zip`
file into the `data/geo_data/flood/europe_flood_data` directory.
This should result in a directory structure that looks like the below:

~~~
data
|-- geo_data
    |-- flood
        |-- europe_flood_data
            |-- data.zip
            |-- readme_river_floods_v1.1.pdf
~~~

Create the `data/geo_data/flood/europe_flood_data/data` directory as below

~~~
mkdir -p ./data/geo_data/flood/europe_flood_data/data
~~~

Unzip the `data.zip` file into the `./data/geo_data/flood/europe_flood_data/data`
directory. This should result in a file structure like below:

~~~
data
|-- geo_data
    |-- flood
        |-- europe_flood_data
            |-- data.zip
            |-- readme_river_floods_v1.1.pdf
            |-- data
                |-- River_discharge_1971_2000_hist.dbf
                |-- River_discharge_1971_2000_hist.prj
                ...
~~~

### Creating output directory

A directory should be created to hold the output of the script.
~~~
mkdir -p ./tmp/load_all_flood
~~~

### Running the script

This script will generate output in `./tmp/load_all_flood`. Examine the script
to see how it does this in more detail. 

Outputs of this script will be a 
mixture of configurations (stored in `./tmp/load_all_flood/conf`), databases
(stored in `./tmp/load_al_flood/databases`), parquet files 
(stored in `./tmp/load_all_flood/parquet`), and visualization files
(stored in (`./tmp/load_all_flood/visualization`))

~~~
python ./examples/loading/flood_data/load_all_flood.py
~~~


