# RAPIDS TPCx-BB

## Overview

TPCx-BB is a Big Data benchmark for enterprises that includes 30 queries representing real-world ETL & ML workflows at various "scale factors": SF1000 is 1 TB of data, SF10000 is 10TB. Each “query” is in fact a model workflow that can include SQL, user-defined functions, careful sub-setting and aggregation, and machine learning. To date, these queries have been run with [Apache Hive](http://hive.apache.org/) and [Apache Spark](http://spark.apache.org/).

This repository provides implementations of the TPCx-BB queries using [RAPIDS](https://rapids.ai/) libraries. For more information about the TPCx-BB Benchmark, please visit the [TPCx-BB homepage](http://www.tpc.org/tpcx-bb/default.asp).


## Conda Environment Setup

We provide a conda environment definition specifying all RAPIDS dependencies needed to run our query implementations. To install and activate it:

```bash
CONDA_ENV="rapids-tpcx-bb"
conda env create --name $CONDA_ENV -f tpcx-bb/conda/rapids-tpcx-bb.yml
conda activate rapids-tpcx-bb
```

For Query 27, we rely on [spacy](https://spacy.io/). To download the necessary language model after activating the Conda environment:

```bash
python -m spacy download en_core_web_sm
````


### Installing RAPIDS TPCxBB Tools
This repository includes a small local module containing utility functions for running the queries. You can install it with the following:

```bash
cd tpcx-bb/tpcx_bb
python -m pip install .

```

This will install a package named `xbb-tools` into your Conda environment. It should look like this:

```bash
conda list | grep xbb
xbb-tools                 0.2                      pypi_0    pypi
```

Note that this Conda environment needs to be replicated or installed manually on all nodes, which will allow starting one dask-cuda-worker per node.


## Cluster Setup

We use the `dask-scheduler` and `dask-cuda-worker` command line interfaces to start a Dask cluster. We provide a `cluster_configuration` directory with a bash script to help you set up an NVLink-enabled cluster using UCX.

Before running the script, you'll need to make a few small changes specific to your machines.

In `cluster_configuration/cluster-startup.sh`, please do the following:

    - Update `INTERFACE=...` to refer to the relevant network interface present on your cluster.
    - Update `CONDA_ENV_PATH=...` to refer to your conda environment path.
    - Update `CONDA_ENV_NAME=...` to refer to the name of the conda environment you created, perhaps using the `yml` files provided in this repository.
    - Update `SCHEDULER=...` to refer to the host name of the node you intend to use as the scheduler.

In `cluster_configuration/cluster-scheduler.json`, please update the scheduler address to be the address for the network interface you chose for `INTERFACE=...` just above. Note that if you are not using UCX, you'll need to adjust the address to be `tcp://...` rather than `ucx://...`.


To start up the cluster, please run the following on every node.

```bash
bash cluster-startup.sh NVLINK
```


## Running the Queries

To run a query, starting from the repository root, go to the `queries` directory:

```bash
cd tpcx_bb/queries/
```

Choose a query to run, and `cd` to that directory. We'll pick query 07.

```bash
cd q07
```

Activate the conda environment you've created.

The queries assume that they can attach to a running Dask cluster. Command line arguments are used to determine the cluster and dataset on which to run the queries. The following is an example of running query 07.

```bash
python tpcx_bb_query_07.py --data_dir=$DATA_DIR --cluster_host=$SCHEDULER_IP --output_dir=$OUTPUT_DIR
```

- `data_dir` points to the location of the data
- `cluster_host` corresponds to the schedyler address of the running Dask cluster
    - In this case, this query would attempt to connect to a cluster running at `$SCHEDULER_IP`, which we configured during cluster startup
- `output_dir` points to where the queries should write output files. If not specified, queries will write output into their respective directories


## Performance Tracking

This repository contains support for performance-tracking automation using Google Sheets. If you install `gspread` and `oauth2client` in your conda environment, you will be able to pass `--sheet` and `--tab` arguments the query CLI to push performance metrics to Google Sheets. If you do not have these libraries installed, you will see `Please install gspread and oauth2client to use Google Sheets automation` after running a query. It's fine to ignore this output.

Expanding on the example above, you could do the following:

```bash
python tpcx_bb_query_07.py --data_dir=$DATA_DIR --cluster_host=$SCHEDULER_IP --output_dir=$OUTPUT_DIR --sheet=TPCx-BB --tab="SF1000 Benchmarking Matrix"
```



### Running all of the Queries

As a convenience, you can sequentially run all the queries with the provided `benchmark_runner.py` script. Instead of passing the command line arguments directly, you will need to fill in the same information in `benchmark_config.yaml`, which can be found in the `benchmark_runner/` directory.


## BlazingSQL

We include BlazingSQL implementations of several queries. As we continue scale testing BlazingSQL implementations, we'll add them to the `queries` folder in the appropriate locations.

We provide a conda environment definition specifying all RAPIDS dependencies needed to run the BSQL query implementations. To install and activate it:

```bash
CONDA_ENV="rapids-bsql-tpcx-bb"
conda env create --name $CONDA_ENV -f tpcx-bb/conda/rapids-bsql-tpcx-bb.yml
conda activate rapids-bsql-tpcx-bb
```

The environment will also require installation of the `xbb_tools` module. Directions are provided [above](#installing-rapids-tpcxbb-tools).


### Cluster Configuration for TCP

BlazingSQL currently supports clusters using TCP. Please follow the instructions above, making sure to use the InfiniBand interface as the `INTERFACE` variable. Then, start the cluster with:

```bash
bash cluster-startup.sh TCP
```

## Data Generation

The RAPIDS queries expect [Apache Parquet](http://parquet.apache.org/) formatted data. We provide a [Jupyter notebook](tpcx_bb/data-conversion.ipynb) which can be used to convert bigBench dataGen's raw CSV files to optimally sized Parquet partitions.
