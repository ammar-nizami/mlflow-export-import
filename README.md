# MLflow Export Import

This package provides tools to export and import MLflow objects (runs, experiments or registered models) from one MLflow tracking server (Databricks workspace) to another.
See the [Databricks MLflow Object Relationships](https://github.com/mlflow/mlflow-resources/blob/master/slides/Databricks_MLflow_Object_Relationships.pdf) slide deck.

## Useful Links
  * [Point tools README](README_point.md)
    * `export_experiment` [API](mlflow_export_import/experiment/export_experiment.py#L25-L30)
    * `export_model` [API](mlflow_export_import/model/export_model.py#L31-L35)
    * `export_run` [API](mlflow_export_import/run/export_run.py#L47-L51)
    * `import_experiment` [API](mlflow_export_import/experiment/import_experiment.py#L31-L35)
    * `import_model` [API](mlflow_export_import/model/import_model.py#L83-L91)
    * `import_run` [API](mlflow_export_import/run/import_run.py#L48-L54)
  * [Bulk tools README](README_bulk.md)
  * [Databricks notebooks for object export/import](databricks_notebooks/README.md).

## Architecture

<img src="architecture.png" height="330" />

## Overview

### Why use MLflow Export Import?
  * Share and collaborate with other data scientists in the same or another tracking server.
    * For example, clone a favorite experiment from another user in your own workspace.
  * Migrate experiments to another tracking server.
    * For example, promote a registered model version (and its associated run) from the development to the production tracking server.
  * Backup your MLflow objects.
  * Disaster recovery.

### Migration modes

|Source tracking server | Destination tracking server | Note |
|-------|------------|---|
| Open source | Open source | common |
| Open source | Databricks | less common |
| Databricks | Databricks |common |
| Databricks | Open source | rare |

### Two migration tool contexts 

* Open source MLflow Python CLI scripts - this page.
* [Databricks notebooks](databricks_notebooks/README.md).


### Two sets of migration tools

* [Point tools](README_point.md). Low-level tools to copy individual MLflow objects and have fine-grained control over the target names.
For example, if you wish to clone an experiment in the same tracking server (workspace), use these tools.
* [Bulk tools](README_bulk.md). High-level tools to copy an entire tracking server or the web of MLflow objects (runs and experiments) associated with registered models. 
Full object referential integrity is maintained as well as the original MLflow object names.
  * For registered models it exports:
    * All the latest versions of a model.
    * The run associated with the version.
    * The experiment that the run belongs to.

### Other
* [Miscellanous tools](README_tools.md) 

## Limitations

### General Limitations

* Nested runs are only supported when you import an experiment. For a run, it is still a TODO.
* If the run linked to a registered model version does not exist (has been deleted) the version is not exported 
  since when importing [MLflowClient.create_model_version](https://mlflow.org/docs/latest/python_api/mlflow.tracking.html#mlflow.tracking.MlflowClient.create_model_version) requires a run ID.

### Databricks Limitations

#### Exporting Notebook Revisions
* The notebook revision associated with the run can be exported. It is stored as an artifact in the `notebooks` folder under the run's `artifacts` root.
*  You can save the notebook in the suppported SOURCE, HTML, JUPYTER and DBC formats. 

#### Importing Notebooks

* Partial functionality due to Databricks REST API limitations.
* The Databricks REST API does not support:
  * Importing a notebook with its entire revision history.
  * Linking an imported run with a given notebook revision.
* When you import a run, the link to its source notebook revision ID will be a dead link and therefore the UI link is broken.
* As a convenience, the import tools allows you to import the exported notebook into Databricks. For more details, see:
  *  [README_point - Import run](README_point.md#Import-run)
  *  [README_point - Import experiment](README_point.md#Import-Experiment)
* The imported notebook cannot be attached to the new run that it is associated with.
* You must export a notebook in the SOURCE format for the notebook to be imported.


#### Used ID
* When importing a run or experiment, for open source MLflow you can specify the user owner. 
* OSS MLflow - the destination run `mlflow.user` tag will be the same as the source `mlflow.user` tag since OSS MLflow allows you to set this tag.
* Databricks MLflow - you cannot set the `mlflow.user` tag.  The `mlflow.user` will be based on the personal access token (PAT) of the importing user.

## Common options details 

`notebook-formats` - If exporting a Databricks run, the run's notebook revision can be saved in the specified formats (comma-delimited argument). Each format is saved in the notebooks folder of the run's artifact root directory as `notebook.{format}`. Supported formats are  SOURCE, HTML, JUPYTER and DBC. See Databricks [Export Format](https://docs.databricks.com/dev-tools/api/latest/workspace.html#notebookexportformat) documentation.

`use-src-user-id` -  Set the destination user ID to the source user ID. Source user ID is ignored when importing into Databricks since the user is automatically picked up from your Databricks access token.

`export-source-tags` - Exports source information under the `mlflow_export_import` tag prefix. See section below for details.

### MLflow Export Import Source Run Tags - `mlflow_export_import`

For governance purposes, original source run information is saved under the `mlflow_export_import` tag prefix. When you import a run, the values of `RunInfo` are auto-generated for you as well as some tags. 

This is useful for governance, provenance and auditing purposes for regulated industries such as finance and HLS (health case and life science) industries.

If the `export-source-tags` option is set on an export tool, three sets of source run tags will be saved under the `mlflow_export_import` prefix.

* **MLflow system tags.** Prefix: `mlflow_export_import.mlflow`. Saves all source MLflow system tags starting with `mlflow.` 
  * For example, the source tag `mlflow.source.type` is saved as the destination tag `mlflow_export_import.mlflow.source.type`.

* **RunInfo field tags.** Prefix: `mlflow_export_import.run_info`. Saves all source [RunInfo](https://mlflow.org/docs/latest/python_api/mlflow.entities.html#mlflow.entities.RunInfo) fields.
  * For example RunInfo.run_id is stored as `mlflow_export_import.run_info.run_id`.
  * Note that since MLflow tag values must be a string, non-string `RunInfo` fields (int) are cast to a string such as `start_time`.

* **Metadata tags.** Prefix: `mlflow_export_import.metadata`.  Tags indicating source export metadata information 
  * Example: `mlflow_export_import.metadata.tracking_uri`.

#### Open Source Mlflow Export Import Tags

See [sample run tags](samples/oss_mlflow/experiments/sklearn_wine/77a09d17edcf47e985403241c576debb/run.json)

##### MLflow system tags

|Tag | Value |
|----|-------|
| mlflow_export_import.mlflow.log-model.history | [{\run_id\: \77a09d17edcf47e985403241c576debb\ | \artifact_path\: \model\, \utc_time_created\: \2022-06-12 03:34:39.289551\, \flavors\: {\python_function\: {\model_path\: \model.pkl\, \loader_module\: \mlflow.sklearn\, \python_version\: \3.7.6\, \env\: \conda.yaml\}, \sklearn\: {\pickled_model\: \model.pkl\, \sklearn_version\: \1.0.2\, \serialization_format\: \cloudpickle\, \code\: null}}, \model_uuid\: \38c43fc59c734b0a80704ac3214ea2c3\, \mlflow_version\: \1.26.1\}, {\run_id\: \77a09d17edcf47e985403241c576debb\, \artifact_path\: \onnx-model\, \utc_time_created\: \2022-06-12 03:34:42.110784\, \flavors\: {\python_function\: {\loader_module\: \mlflow.onnx\, \python_version\: \3.7.6\, \data\: \model.onnx\, \env\: \conda.yaml\}, \onnx\: {\onnx_version\: \1.10.2\, \data\: \model.onnx\, \providers\: [\CUDAExecutionProvider\, \CPUExecutionProvider\], \code\: null}}, \model_uuid\: \ddf79625e4d241b7813e601f31b1222f\, \mlflow_version\: \1.26.1\}],
| mlflow_export_import.mlflow.runName | train.sh |
| mlflow_export_import.mlflow.source.git.commit | 67fb8f823ec794902cdbb67be653a6155a0b5172 |
| mlflow_export_import.mlflow.source.name | /Users/andre/git/mlflow-examples/python/sklearn/wine_quality/train.py |
| mlflow_export_import.mlflow.source.type | LOCAL |
| mlflow_export_import.mlflow.user | andre |

##### MLflow RunInfo tags

|Tag | Value |
|----|-------|
| mlflow_export_import.run_info.artifact_uri | /opt/mlflow/server/mlruns/2/77a09d17edcf47e985403241c576debb/artifacts |
| mlflow_export_import.run_info.end_time | 1655004883611 |
| mlflow_export_import.run_info.experiment_id | 2 |
| mlflow_export_import.run_info.lifecycle_stage | active |
| mlflow_export_import.run_info.run_id | 77a09d17edcf47e985403241c576debb |
| mlflow_export_import.run_info.start_time | 1655004878844 |
| mlflow_export_import.run_info.status | FINISHED |
| mlflow_export_import.run_info.user_id | andre |

##### Metadata tags

|Tag | Value | Description |
|----|-------|-------------|
| mlflow_export_import.metadata.experiment_name | sklearn_wine | Name of experiment |
| mlflow_export_import.metadata.timestamp | 1655007510 | Time when run was exported |
| mlflow_export_import.metadata.timestamp_nice | 2022-06-12 04:18:30 | ibid |
| mlflow_export_import.metadata.tracking_uri | http://127.0.0.1:5020 | Source tracking server URI |

### Databricks MLflow source tags

See [sample run tags](samples/databricks/experiments/sklearn_wine/f2e3f75c845d4365addbc9c0262a58a5/run.json).

##### MLflow system tags

|Tag | Value | 
|----|-------|
| mlflow_export_import.mlflow.databricks.cluster.id         | 0318-151752-abed99                                                                                             |
| mlflow_export_import.mlflow.databricks.cluster.info       | {"cluster_name":"Shared Autoscaling Americas","spark_version":"10.2.x-cpu-ml-scala2.12","node_type_id":"i3.2xl |
| mlflow_export_import.mlflow.databricks.cluster.libraries  | {"installable":[],"redacted":[]}                                                                               |
| mlflow_export_import.mlflow.databricks.notebook.commandID | 6101304639030907941_7207589773925520000_e458c0ed7c5c4e52b020b1b92d39b308                                       |
| mlflow_export_import.mlflow.databricks.notebookID         | 3532228                                                                                                        |
| mlflow_export_import.mlflow.databricks.notebookPath       | /Users/andre@mycompany.com/mlflow/02a_Sklearn_Train_Predict    |
| mlflow_export_import.mlflow.databricks.notebookRevisionID | 1647395473565                                                                                                  |
| mlflow_export_import.mlflow.databricks.webappURL          | https://demo.cloud.databricks.com                                                                              |
| mlflow_export_import.mlflow.log-model.history             | [{"artifact_path":"sklearn-model","signature":{"inputs":"[{\"name\": \"fixed acidity\", \"type\": \"double\"}, |
| mlflow_export_import.mlflow.runName                       | sklearn                                                                                                        |
| mlflow_export_import.mlflow.source.name                   | /Users/andre@mycompany.com/mlflow/02a_Sklearn_Train_Predict    |
| mlflow_export_import.mlflow.source.type                   | NOTEBOOK                                                                                                       |
| mlflow_export_import.mlflow.user                          | andre@mycompany.com                                                                                 |


##### MLflow RunInfo tags

|Tag | Value | 
|----|-------|
| mlflow_export_import.run_info.artifact_uri                | dbfs:/databricks/mlflow/3532228/826a19c33d8c461ebf91aa90c25a5dd8/artifacts |
| mlflow_export_import.run_info.end_time                    | 1647395473410                                                              |
| mlflow_export_import.run_info.experiment_id               | 3532228                                                                    |
| mlflow_export_import.run_info.lifecycle_stage             | active                                                                     |
| mlflow_export_import.run_info.run_id                      | 826a19c33d8c461ebf91aa90c25a5dd8                                           |
| mlflow_export_import.run_info.run_uuid                    | 826a19c33d8c461ebf91aa90c25a5dd8                                           |
| mlflow_export_import.run_info.start_time                  | 1647395462575                                                              |
| mlflow_export_import.run_info.status                      | FINISHED                                                                   |
| mlflow_export_import.run_info.user_id                     |                                                                            |

##### Metadata tags
|Tag | Value | 
|----|-------|
| mlflow_export_import.metadata.experiment_name             | /Users/andre@mycompany.com/mlflow/02a_Sklearn_Train_Predict    |
| mlflow_export_import.metadata.timestamp                   | 1655148303 |
| mlflow_export_import.metadata.timestamp_nice              | 2022-06-13 19:25:03 |
| mlflow_export_import.metadata.tracking_uri                | databricks |

## Setup

Supports python 3.7.6 or above.


### Local setup

First create a virtual environment.
```
python -m venv mlflow-export-import
source mlflow-export-import/bin/activate
```

There are two different ways to install the package.

#### 1. Install from github directly

```
pip install git+https:///github.com/mlflow/mlflow-export-import/#egg=mlflow-export-import
```

#### 2. Install from github clone
```
git clone https://github.com/mlflow/mlflow-export-import
cd mlflow-export-import
pip install -e .
```

### Databricks setup

There are two different ways to install the package.

#### 1. Install package in notebook

[Install notebook-scoped libraries with %pip](https://docs.databricks.com/libraries/notebooks-python-libraries.html#install-notebook-scoped-libraries-with-pip).


```
pip install git+https:///github.com/mlflow/mlflow-export-import/#egg=mlflow-export-import
```

#### 2. Install package as a wheel on cluster

Build the wheel artifact, upload it to DBFS and then [install it on your cluster](https://docs.databricks.com/libraries/cluster-libraries.html).

```
python setup.py bdist_wheel
databricks fs cp dist/mlflow_export_import-1.0.0-py3-none-any.whl {MY_DBFS_PATH}
```

### Databricks MLflow usage

To run the tools externally (from your laptop) against a Databricks tracking server (workspace) set the following environment variables.
```
export MLFLOW_TRACKING_URI=databricks
export DATABRICKS_HOST=https://mycompany.cloud.databricks.com
export DATABRICKS_TOKEN=MY_TOKEN
```
For full details see [Access the MLflow tracking server from outside Databricks](https://docs.databricks.com/applications/mlflow/access-hosted-tracking-server.html).


## Running tools

The main tool scripts can be executed either as a standard Python script or console script.

Python [console scripts](https://python-packaging.readthedocs.io/en/latest/command-line-scripts.html#the-console-scripts-entry-point)  (such as export-run, import-run, etc.) are provided as a convenience. For a list of scripts see [setup.py](setup.py).

This allows you to use:
```
export-experiment --help
```
instead of:
```
python -u -m mlflow_export_import.experiment.export_experiment --help
```
